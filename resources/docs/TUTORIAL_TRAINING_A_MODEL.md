# Tutorial 5: Training a Model

This part of the tutorial shows how you can train your own sequence labeling and text
classification models using state-of-the-art word embeddings.

For this tutorial, we assume that you're familiar with the [base types](/resources/docs/TUTORIAL_BASICS.md) of this
library and how [word embeddings](/resources/docs/TUTORIAL_WORD_EMBEDDING.md) work.


## Reading A Column-Formatted Dataset

Most sequence labeling datasets in NLP use some sort of column format in which each line is a word and each column is
one level of linguistic annotation. See for instance this sentence:

```console
George N B-PER
Washington N I-PER
went V O
to P O
Washington N B-LOC
```

The first column is the word itself, the second coarse PoS tags, and the third BIO-annotated NER tags. To read such a dataset,
define the column structure as a dictionary and use a helper method.

```python

# define columns
columns = {0: 'text', 1: 'pos', 2: 'np'}

# this is the folder in which train, test and dev files reside
data_folder = '/path/to/data/folder'

# retrieve corpus using column format, data folder and the names of the train, dev and test files
corpus: TaggedCorpus = NLPTaskDataFetcher.fetch_column_corpus(data_folder, columns,
                                                              train_file='train.txt',
                                                              test_file='test.txt',
                                                              dev_file='dev.txt')
```

This gives you a `TaggedCorpus` object that contains the train, dev and test splits, each as a list of `Sentence`.
So, to check how many sentences there are in the training split, do

```python
len(corpus.train)
```

You can also access a sentence and check out annotations. Lets assume that the first sentence in the training split is
the example sentence from above, then executing these commands

```python
print(corpus.train[0].to_tagged_string('pos'))
print(corpus.train[0].to_tagged_string('ner'))
```

will print the sentence with different layers of annotation:

```console
George <N> Washington <N> went <V> to <P> Washington <N>

George <B-PER> Washington <I-PER> went to Washington <B-LOC> .
```


## The TaggedCorpus Object

The `TaggedCorpus` contains a bunch of useful helper functions. For instance, you can downsample the data by calling
`downsample()` and passing a ratio. So, if you normally get a corpus like this:

```python
original_corpus = NLPTaskDataFetcher.fetch_data(NLPTask.CONLL_03)
```

then you can downsample the corpus, simply like this:

```python
downsampled_corpus = NLPTaskDataFetcher.fetch_data(NLPTask.CONLL_03).downsample(0.1)
```

If you print both corpora, you see that the second one has been downsampled to 10% of the data.

```python
print("--- 1 Original ---")
print(original_corpus)

print("--- 2 Downsampled ---")
print(downsampled_corpus)
```

This should print:

```console
--- 1 Original ---
TaggedCorpus: 14987 train + 3466 dev + 3684 test sentences

--- 2 Downsampled ---
TaggedCorpus: 1499 train + 347 dev + 369 test sentences
```

## Training a Sequence Labeling Model

Here is example code for a small NER model trained over CoNLL-03 data, using simple GloVe embeddings.
In this example, we downsample the data to 10% of the original data.

```python
from flair.data import TaggedCorpus
from flair.data_fetcher import NLPTaskDataFetcher, NLPTask
from flair.embeddings import TokenEmbeddings, WordEmbeddings, StackedEmbeddings
from typing import List

# 1. get the corpus
corpus: TaggedCorpus = NLPTaskDataFetcher.fetch_data(NLPTask.CONLL_03).downsample(0.1)
print(corpus)

# 2. what tag do we want to predict?
tag_type = 'ner'

# 3. make the tag dictionary from the corpus
tag_dictionary = corpus.make_tag_dictionary(tag_type=tag_type)
print(tag_dictionary.idx2item)

# 4. initialize embeddings
embedding_types: List[TokenEmbeddings] = [

    WordEmbeddings('glove'),

    # comment in this line to use character embeddings
    # CharacterEmbeddings(),

    # comment in these lines to use contextual string embeddings
    # CharLMEmbeddings('news-forward'),
    # CharLMEmbeddings('news-backward'),
]

embeddings: StackedEmbeddings = StackedEmbeddings(embeddings=embedding_types)

# 5. initialize sequence tagger
from flair.models import SequenceTagger

tagger: SequenceTagger = SequenceTagger(hidden_size=256,
                                        embeddings=embeddings,
                                        tag_dictionary=tag_dictionary,
                                        tag_type=tag_type,
                                        use_crf=True)

# 6. initialize trainer
from flair.trainers import SequenceTaggerTrainer

trainer: SequenceTaggerTrainer = SequenceTaggerTrainer(tagger, corpus)

# 7. start training
trainer.train('resources/taggers/example-ner',
              learning_rate=0.1,
              mini_batch_size=32,
              max_epochs=150)

# 8. plot training curves (optional)
from flair.visual.training_curves import Plotter
plotter = Plotter()
plotter.plot_training_curves('resources/taggers/example-ner/loss.tsv')
plotter.plot_weights('resources/taggers/example-ner/weights.txt')

```

Alternatively, try using a stacked embedding with charLM and glove, over the full data, for 150 epochs.
This will give you the state-of-the-art accuracy we report in the paper. To see the full code to reproduce experiments,
check [here](/resources/docs/EXPERIMENTS.md).

## Training a Text Classification Model

Here is example code for training a text classifier over the AGNews corpus, using  a combination of simple GloVe
embeddings and contextual string embeddings. In this example, we downsample the data to 10% of the original data.

The AGNews corpus can be downloaded [here](https://www.di.unipi.it/~gulli/AG_corpus_of_news_articles.html).

### Preparing the data

We use the [FastText format](https://fasttext.cc/docs/en/supervised-tutorial.html) for text classification data, in which
each line in the file represents a text document. A document can have one or multiple labels that are defined at the beginning of the line starting with the prefix
`__label__`. This looks like this:


```bash
__label__<label_1> <text>
__label__<label_1> __label__<label_2> <text>
```

Point the `NLPTaskDataFetcher` to this file to convert each line to a `Sentence` object annotated with the labels. It returns a list of `Sentence`.

```python
from flair.data_fetcher import NLPTaskDataFetcher

# use your own data path
data_folder = 'path/to/text-classification/formatted/data'

# get training, test and dev data
sentences: List[Sentence] = NLPTaskDataFetcher.read_text_classification_file(data_folder)
```


### Training the classifier

To train a model, you need to create three files in this way: A train, dev and test file. After you converted the data you can use the
`NLPTaskDataFetcher` to read the data, if the data files are located in the following folder structure:
```
/resources/tasks/ag_news/train.txt
/resources/tasks/ag_news/dev.txt
/resources/tasks/ag_news/test.txt
```

Here the example code for training the text classifier.
```python
from flair.data import TaggedCorpus
from flair.data_fetcher import NLPTaskDataFetcher, NLPTask
from flair.embeddings import WordEmbeddings, CharLMEmbeddings, DocumentLSTMEmbeddings
from flair.models.text_classification_model import TextClassifier
from flair.trainers.text_classification_trainer import TextClassifierTrainer

# 1. get the corpus
corpus: TaggedCorpus = NLPTaskDataFetcher.fetch_data(NLPTask.AG_NEWS).downsample(0.1)

# 2. create the label dictionary
label_dict = corpus.make_label_dictionary()

# 3. make a list of word embeddings
word_embeddings = [WordEmbeddings('glove'),
                   CharLMEmbeddings('news-forward'),
                   CharLMEmbeddings('news-backward')]

# 4. init document embedding by passing list of word embeddings
document_embeddings: DocumentLSTMEmbeddings = DocumentLSTMEmbeddings(word_embeddings,
                                                                     hidden_states=512,
                                                                     reproject_words=True,
                                                                     reproject_words_dimension=256,)

# 5. create the text classifier
classifier = TextClassifier(document_embeddings, label_dictionary=label_dict, multi_label=False)

# 6. initialize the text classifier trainer
trainer = TextClassifierTrainer(classifier, corpus, label_dict)

# 7. start the trainig
trainer.train('resources/ag_news/results',
              learning_rate=0.1,
              mini_batch_size=32,
              anneal_factor=0.5,
              patience=5,
              max_epochs=150)

# 8. plot training curves (optional)
from flair.visual.training_curves import Plotter
plotter = Plotter()
plotter.plot_training_curves('resources/ag_news/results/loss.tsv')
plotter.plot_weights('resources/ag_news/results/weights.txt')
```

Once the model is trained you can use it to predict the class of new sentences. Just call the `predict` method of the
model.

```python
sentences = model.predict(Sentence('France is the current world cup winner.'))

```
The predict method adds the class labels directly to the sentences. Each label has a name and a confidence value.
```python
for sentence in sentences:
    print(sentence.labels)
```


## Plotting Training Curves and Weights

Flair includes a helper method to plot training curves and weights in the neural network.
Both the `SequenceTaggerTrainer` and `TextClassificationTrainer` automatically generate a `loss.tsv` and a `weights.txt`
file in the result folder.

After training, simple point the plotter to these files:

```python
from flair.visual.training_curves import Plotter
plotter = Plotter()
plotter.plot_training_curves('resources/ag_news/results/loss.tsv')
plotter.plot_weights('resources/ag_news/results/weights.txt')
```

This generates PNG plots in the result folder.


## Scalability: Training on Large Data Sets

The main thing to consider when using `CharLMEmbeddings` (which you should) is that they are
somewhat costly to generate for large training data sets. Depending on your setup, you can
set options to optimize training time. There are three questions to ask:

1. Do you have a GPU?

`CharLMEmbeddings` are generated using Pytorch RNNs and are thus optimized for GPUs. If you have one,
you can set large mini-batch sizes to make use of batching. If not, you may want to use smaller language models.
For English, we package 'fast' variants of our embeddings, loadable like this: `CharLMEmbeddings('news-forward-fast')`.

Regardless, all computed embeddings get materialized to disk upon first computation. This means that if you rerun an
experiment on the same dataset, they will be retrieved from disk instead of re-computed, potentially saving a lot
of time.

2. Do embeddings for the entire dataset fit into memory?

In the best-case scenario, all embeddings for the dataset fit into your regular memory, which greatly increases
training speed. If this is not the case, you must set the flag `embeddings_in_memory=False` in the respective trainer
 (i.e. `SequenceTaggerTrainer` or  `TextClassifierTrainer`) to
avoid memory problems. With the flag, embeddings are either (a) recomputed at each epoch or (b)
retrieved from disk (where they are materialized by default). The second option is the default and is typically
much faster.

3. Do you have a fast hard drive?

You benefit the most from the default behavior of storing computed embeddings on disk for later retrieval
if your disk is large and fast. If you either do not have a lot of disk space, or a really slow hard drive,
you should disable this option. You can do this when instantiating the embeddings by setting `use_cache=False`. So
instantiate like this: `CharLMEmbeddings('news-forward-fast', use_cache=False')`



## Next

You can now look into [training your own embeddings](/resources/docs/TUTORIAL_TRAINING_LM_EMBEDDINGS.md).