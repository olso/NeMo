Tutorial
========

In this tutorial, we are going to implement a joint intent and slot filling system with pretrained BERT model based on `BERT for Joint Intent Classification and Slot Filling <https://arxiv.org/abs/1902.10909>`_ :cite:`chen2019bert`. All code used in this tutorial is based on ``examples/nlp/joint_intent_slot_with_bert.py``.

There are four pretrained BERT models that we can select from using the argument `--pretrained_bert_model`. We're currently using the script for loading pretrained models from `pytorch_transformers`. See the list of available pretrained models `here <https://huggingface.co/pytorch-transformers/pretrained_models.html>`__. 


Preliminaries
-------------

**Model details**
This model jointly train the sentence-level classifier for intents and token-level classifier for slots by minimizing the combined loss of the two classifiers:

        intent_loss * intent_loss_weight + slot_loss * (1 - intent_loss_weight)

When `intent_loss_weight = 0.5`, this loss jointly maximizes:

        p(y | x)P(s1, s2, ..., sn | x)

with x being the sequence of n tokens (x1, x2, ..., xn), y being the predicted intent for x, and s1, s2, ..., sn being the predicted slots corresponding to x1, x2, ..., xn.

**Datasets.** 

This model can work with any dataset that follows the format:
    * input file: a `tsv` file with the first line as a header [sentence][tab][label]

    * slot file: slot labels for all tokens in the sentence, separated by space. The length of the slot labels should be the same as the length of all tokens in sentence in input file.

Currently, the datasets that we provide pre-processing script for include ATIS which can be downloaded from `Kaggle <https://www.kaggle.com/siddhadev/atis-dataset-from-ms-cntk>`_ and the SNIPS spoken language understanding research dataset which can be requested from `here <https://github.com/snipsco/spoken-language-understanding-research-datasets>`__. You can find the pre-processing script in ``collections/nemo_nlp/nemo_nlp/text_data_utils.py``.


Code structure
--------------

First, we instantiate Neural Module Factory which defines 1) backend (PyTorch or TensorFlow), 2) mixed precision optimization level, 3) local rank of the GPU, and 4) an experiment manager that creates a timestamped folder to store checkpoints, relevant outputs, log files, and TensorBoard graphs.

    .. code-block:: python

        nf = nemo.core.NeuralModuleFactory(
                        backend=nemo.core.Backend.PyTorch,
                        local_rank=args.local_rank,
                        optimization_level=args.amp_opt_level,
                        log_dir=work_dir,
                        create_tb_writer=True,
                        files_to_copy=[__file__])

We define tokenizer which transforms text into BERT tokens, using a built-in tokenizer by `pytorch_transformers`. This will tokenize text following the mapping of the original BERT model.

    .. code-block:: python

        from pytorch_transformers import BertTokenizer
        tokenizer = BertTokenizer.from_pretrained(args.pretrained_bert_model)

Next, we define all Neural Modules participating in our joint intent slot filling classification pipeline.
    
    * Data layer: converting from the formatted dataset to data loader that feeds data into our model.

    .. code-block:: python

        data_layer = nemo_nlp.BertJointIntentSlotDataLayer(
                path_to_data=data_file,
                path_to_slot=slot_file,
                pad_label=pad_label,
                tokenizer=tokenizer,
                mode=mode,
                max_seq_length=max_seq_length,
                num_samples=num_samples,
                batch_size=batch_size,
                shuffle=shuffle,
                num_workers=0,
                local_rank=local_rank
            )

        ids, type_ids, input_mask, slot_mask, intents, slots = data_layer()


    * Load the pretrained model and get the hidden states for the corresponding inputs.

    .. code-block:: python

        hidden_states = pretrained_bert_model(input_ids=ids,
                                              token_type_ids=type_ids,
                                              attention_mask=input_mask)


    * Create the classifier heads for our task.

    .. code-block:: python

        classifier = nemo_nlp.JointIntentSlotClassifier(
                                        hidden_size=hidden_size,
                                        num_intents=num_intents,
                                        num_slots=num_slots,
                                        dropout=args.fc_dropout)

        intent_logits, slot_logits = classifier(hidden_states=hidden_states)


    * Create loss function

    .. code-block:: python

        loss_fn = nemo_nlp.JointIntentSlotLoss(num_slots=num_slots)

        loss = loss_fn(intent_logits=intent_logits,
                       slot_logits=slot_logits,
                       input_mask=input_mask,
                       intents=intents,
                       slots=slots)


    * Create relevant callbacks for saving checkpoints, printing training progresses and evaluating results

    .. code-block:: python

        if mode == 'train':
            callback_fn = nemo.core.SimpleLossLoggerCallback(
                tensors=[loss, intent_logits, slot_logits],
                print_func=lambda x: str(np.round(x[0].item(), 3)),
                tb_writer=exp.tb_writer,
                get_tb_values=lambda x: [["loss", x[0]]],
                step_freq=100)
        elif mode == 'eval':
            callback_fn = nemo.core.EvaluatorCallback(
                eval_tensors=[intent_logits, slot_logits, intents, slots],
                user_iter_callback=lambda x, y: eval_iter_callback(
                    x, y, data_layer),
                user_epochs_done_callback=lambda x: eval_epochs_done_callback(
                    x, f'{exp.work_dir}/graphs'),
                tb_writer=exp.tb_writer,
                eval_step=steps_per_epoch)


        ckpt_callback = nemo.core.CheckpointCallback(
            folder=exp.ckpt_dir,
            epoch_freq=args.save_epoch_freq,
            step_freq=args.save_step_freq)


    * Finally, we define the optimization parameters and run the whole pipeline.

    .. code-block:: python

        lr_policy_fn = get_lr_policy(args.lr_policy,
                                     total_steps=args.num_epochs * steps_per_epoch,
                                     warmup_ratio=args.lr_warmup_proportion)
        nf.train(tensors_to_optimize=[train_loss],
             callbacks=[callback_train, callback_eval, ckpt_callback],
             lr_policy=lr_policy_fn,
             optimizer=args.optimizer_kind,
             optimization_params={"num_epochs": num_epochs,
                                  "lr": args.lr,
                                  "weight_decay": args.weight_decay})

Model training
--------------

To train a joint intent slot filling model, run ``joint_intent_slot_with_bert.py`` located at ``nemo/examples/nlp``:

    .. code-block:: python

        python -m torch.distributed.launch --nproc_per_node=2 joint_intent_slot_with_bert.py \
            --data_dir <path to data>
            --work_dir <where you want to log your experiment> \
            --max_seq_length \
            --optimizer_kind 
            ...

To do inference, run:

    .. code-block:: python

        python -m joint_intent_slot_infer.py \
            --data_dir <path to data> \
            --work_dir <path to checkpoint folder>


To do inference on a single query, run:
    
    .. code-block:: python

        python -m joint_intent_slot_infer.py \
            --work_dir <path to checkpoint folder>
            --query <query>


References
----------

.. bibliography:: joint_intent_slot.bib
    :style: plain
