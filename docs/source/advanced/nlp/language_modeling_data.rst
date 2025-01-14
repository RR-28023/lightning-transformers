Language Modeling using Custom Data Processing
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Below we show how to override data processing logic.

Extend the ``LanguageModelingDataModule`` base class
""""""""""""""""""""""""""""""""""""""""""""""""""""

The base data module can be used to modify this code, and follows a simple pattern. Internally the dataset is loaded via HuggingFace Datasets, which returns an `Apache Arrow Parquet <https://arrow.apache.org/docs/python/generated/pyarrow.parquet.ParquetDataset.html>`_ Dataset. This data format is easy to transform and modify using map functions, which you'll see within the class.

.. code-block:: python

    class LanguageModelingDataModule(HFDataModule):

        def __init__(self, cfg: LanguageModelingDataConfig = LanguageModelingDataConfig()):
            super().__init__(cfg=cfg)

        def process_data(self, dataset: Dataset, stage: Optional[str] = None) -> Dataset:
            # `process_data` converting the dataset into features.
            # The dataset is pre-loaded using `load_dataset`.
            ...
            return dataset

        @staticmethod
        def tokenize_function(
            examples,
            tokenizer: Union[PreTrainedTokenizerBase],
            text_column_name: str = None,
        ):
            # tokenizes the data in a specific column using the AutoTokenizer,
            # called by `process_data`
            return tokenizer(examples[text_column_name])

        @staticmethod
        def convert_to_features(examples, block_size: int = None):
            # `process_data` calls this function to convert samples in the dataset into features
            ...

        @property
        def collate_fn(self) -> Callable:
            # `Describes how to collate the samples for the batch given to the model`
            return default_data_collator



Extend ``LanguageModelingDataModule``, like this.

.. code-block:: python

    from lightning_transformers.task.nlp.language_modeling import LanguageModelingDataModule

    class MyLanguageModelingDataModule(LanguageModelingDataModule):
        ...

Make any changes you'd like to the dataset processing via the hooks.

Below we have the pseudo code version to show where most of the changes happened within the hooks:

.. code-block:: python

    from functools import partial

    from datasets import Dataset, Optional
    from transformers import PreTrainedTokenizerBase

    from lightning_transformers.core.nlp.huggingface import HFTransformerDataConfig
    from lightning_transformers.task.nlp.language_modeling import LanguageModelingDataModule


    class MyLanguageModelingDataModule(LanguageModelingDataModule):

        def __init__(self, cfg: HFTransformerDataConfig, tokenizer: PreTrainedTokenizerBase):
            super().__init__(cfg, tokenizer)
            self.tokenized_condition_term = tokenizer("This is a story: ")

        def process_data(self, dataset: Dataset, stage: Optional[str] = None) -> Dataset:
            ...
            # Pass in our additional condition term when converting to features
            convert_to_features = partial(
                self.convert_to_features,
                block_size=self.effective_block_size,
                tokenized_condition_term=self.tokenized_condition_term
            )
            ...
            return dataset

        @staticmethod
        def convert_to_features(examples, block_size: int, **kwargs):
            # Our argument is passed in via kwargs
            tokenized_condition_term = kwargs['tokenized_condition_term']

            ...
            # Add the term to the tokenized blocks of text
            result = {
                k: [tokenized_condition_term + t[i:i + block_size] for i in range(0, total_length, block_size)]
                for k, t in concatenated_examples.items()
            }
            result["labels"] = result["input_ids"].copy()
            return result
