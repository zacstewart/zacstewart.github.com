---
layout: post
title: "Using scikit-learn Pipelines and FeatureUnions"
---
Since I posted a [postmortem][postmortem] of my entry to Kaggle's See Click Fix competition, I've meant to keep sharing things that I learn as I improve my machine learning skills. One that I've been meaning to share is [scikit-learn][sklearn]'s pipeline module. The following is a moderately detailed explanation and a few examples of how I use pipelining when I work on competitions.

The pipeline module of scikit-learn allows you to chain transformers and estimators together in such a way that you can use them as a single unit. This comes in very handy when you need to jump through a few hoops of data extraction, transformation, normalization, and finally train your model (or use it to generate predictions). 

When I first started participating in Kaggle competitions, I would invariably get started with some code that looked similar to this:

```python
train = read_file('data/train.tsv')
train_y = extract_targets(train)
train_essays = extract_essays(train)
train_tokens = get_tokens(train_essays)
train_features = extract_feactures(train)
classifier = MultinomialNB()

scores = []
train_idx, cv_idx in KFold():
  classifier.fit(train_features[train_idx], train_y[train_idx])
  scores.append(model.score(train_features[cv_idx], train_y[cv_idx]))

print("Score: {}".format(np.mean(scores)))
```

Often, this would yield a pretty decent score for a first submission. To improve my ranking on the leaderboard, I would try extracting some more features from the data. Let's say in instead of text n-gram counts, I wanted [tf–idf][tfidf]. In addition, I wanted to include overall essay length. I might as well throw in misspelling counts while I'm at it. Well, I can just tack those into the implementation of `extract_features`. I'd extract three matrices of features–one for each of those ideas and then concatenate them along axis 1. Easy.

After exhausting all the ideas I had for extracting features, I'd find myself normalizing or scaling them, and then realizing that I want to scale some features and normalize others. Before long, it would be 2am and I'd be trying to keep track of matrix column indices by hand so that I can try something out real quick. My code would be a big ball of spaghetti, but somehow I could still understand it, so long as it was all loaded into working memory. I'd cross validate my model again, only to realize that it hurt my score, so never mind on that last idea. Maybe I should fiddle with some of the hyperparameters or something, until finally I `git commit -am "WIP going to bed"`.

# Pipelines

Using a [Pipeline][pipeline] simplifies this process. Instead of manually running through each of these steps, and then tediously repeating them on the test set, you get a nice, declarative interface where it's easy to see the entire model. This example extracts the text documents, tokenizes them, counts the tokens, and then performs a tf–idf transformation before passing the resulting features along to a multinomial naive Bayes classifier:

```python
model = Pipeline([
  ('extract_essays', EssayExractor()),
  ('counts', CountVectorizer()),
  ('tf_idf', TfidfTransformer()),
  ('classifier', MultinomialNB())
])

train = read_file('data/train.tsv')
train_y = extract_targets(train)
scores = []
train_idx, cv_idx in KFold():
  model.fit(train[train_idx], train_y[train_idx])
  scores.append(model.score(train[cv_idx], train_y[cv_idx]))

print("Score: {}".format(np.mean(scores)))
```

This pipeline has what I think of as a linear shape. The data flows straight through each step, until it reaches the classifier.

![Simple scikit-learn Pipeline](/images/pipelines-of-featureunions-of-pipelines/simple-pipeline.svg)

# FeatureUnions

Like I said before, I usually want to extract more features, and that means parallel processes that need to be performed with the data before putting the results together. Using a [FeatureUnion][featureunion], you can model these parallel processes, which are often Pipelines themselves:

```python
pipeline = Pipeline([
  ('extract_essays', EssayExractor()),
  ('features', FeatureUnion([
    ('ngram_tf_idf', Pipeline([
      ('counts', CountVectorizer()),
      ('tf_idf', TfidfTransformer())
    ])),
    ('essay_length', LengthTransformer()),
    ('misspellings', MispellingCountTransformer())
  ])),
  ('classifier', MultinomialNB())
])
```

This example feeds the output of the `extract_essays` step into each of the `ngram_tf_idf`, `essay_length`, and `misspellings` steps and concatenates their outputs (along axis 1) before feeding it into the classifier.

![scikit-learn Pipeline with a FeatureUnion](/images/pipelines-of-featureunions-of-pipelines/featureunion-pipelines.svg)

Usually, I end up with several layers of nested Pipelines and FeatureUnions. For example, here's a crazy Pipeline I pulled from a project I am currently working on:

```python
pipeline = Pipeline([
    ('features', FeatureUnion([
        ('continuous', Pipeline([
            ('extract', ColumnExtractor(CONTINUOUS_FIELDS)),
            ('scale', Normalizer())
        ])),
        ('factors', Pipeline([
            ('extract', ColumnExtractor(FACTOR_FIELDS)),
            ('one_hot', OneHotEncoder(n_values=5)),
            ('to_dense', DenseTransformer())
        ])),
        ('weekday', Pipeline([
            ('extract', DayOfWeekTransformer()),
            ('one_hot', OneHotEncoder()),
            ('to_dense', DenseTransformer())
        ])),
        ('hour_of_day', HourOfDayTransformer()),
        ('month', Pipeline([
            ('extract', ColumnExtractor(['datetime'])),
            ('to_month', DateTransformer()),
            ('one_hot', OneHotEncoder()),
            ('to_dense', DenseTransformer())
        ])),
        ('growth', Pipeline([
            ('datetime', ColumnExtractor(['datetime'])),
            ('to_numeric', MatrixConversion(int)),
            ('regression', ModelTransformer(LinearRegression()))
        ]))
    ])),
    ('estimators', FeatureUnion([
        ('knn', ModelTransformer(KNeighborsRegressor(n_neighbors=5))),
        ('gbr', ModelTransformer(GradientBoostingRegressor())),
        ('dtr', ModelTransformer(DecisionTreeRegressor())),
        ('etr', ModelTransformer(ExtraTreesRegressor())),
        ('rfr', ModelTransformer(RandomForestRegressor())),
        ('par', ModelTransformer(PassiveAggressiveRegressor())),
        ('en', ModelTransformer(ElasticNet())),
        ('cluster', ModelTransformer(KMeans(n_clusters=2)))
    ])),
    ('estimator', KNeighborsRegressor())
])
```

# Custom Transformers

Many of the steps in the previous examples include transformers that don't come with scikit-learn. The ColumnExtractor, DenseTransformer, and ModelTransformer, to name a few, are all custom transformers that I wrote. A transformer is just an object that responds to `fit`, `transform`, and `fit_transform`. This includes built-in transformers (like MinMaxScaler), Pipelines, FeatureUnions, and of course, plain old Python objects that implement those methods. Inheriting from TransformerMixin is not required, but helps to communicate intent, and gets you `fit_transform` for free.

A transformer can be thought of as a data in, data out black box. Generally, they accept a matrix as input and return a matrix of the same shape as output. That makes it easy to reorder and remix them at will. However, I often use [Pandas][pandas] DataFrames, and expect one as input to a transformer. For example, the ColumnExtractor is for extracting columns from a DataFrame.

Sometimes transformers are very simple, like HourOfDayTransformer, which just extracts the hour components out of a vector of datetime objects. Such transformers are "stateless"–they don't need to be fitted, so `fit` is a no-op:

```python
class HourOfDayTransformer(TransformerMixin):

    def transform(self, X, **transform_params):
        hours = DataFrame(X['datetime'].apply(lambda x: x.hour))
        return hours

    def fit(self, X, y=None, **fit_params):
        return self
```

However, sometimes transformers do need to be fitted. Let's take a look at my ModelTransformer. I use this one to wrap a scikit-learn model and make it behave like a transformer. I find these useful when I want to use something like a KMeans clustering model to generate features for another model. It needs to be fitted in order to train the model it wraps.


```python
class ModelTransformer(TransformerMixin):

    def __init__(self, model):
        self.model = model

    def fit(self, *args, **kwargs):
        self.model.fit(*args, **kwargs)
        return self

    def transform(self, X, **transform_params):
        return DataFrame(self.model.predict(X))
```

The pipeline treats these objects like any of the built-in transformers and `fit`s them during the training phase, and `transform`s the data using each one when predicting.

# Final Thoughts

While the initial investment is higher, designing my projects this way ensures that I can continue to adapt and improve it without pulling my hair out keeping all the steps straight. It really starts to pay off when you get into hyperparameter tuning, but I'll save that for another post.

Pipelines unfortunately do not support the `fit_partial` API for out-of-core training. This makes them less useful for large scale or online learning models. I addressed some of this in [my talk on building a language identifier][langue], wherein I trained a model on entire Wikipedia dumps. Unfortunately, it's not as easy as it sounds to make Pipelines support it. The scikit-learn team will probably have to come up with a different pipelining scheme for incremental learning.

If I could add one enhancement to this design, it would be a way to add post-processing steps to the pipeline. I usually end up needing to take a few final steps like setting all negative predictions to `0.0`. The last step in the pipeline is assumed to be the final estimator, and as such the pipeline calls `predict` on it instead of `transform`. Because of this limitation, I end up having to do my post-processing outside of the pipeline:

```python
pipeline.fit(x, y)
predicted = pipeline.predict(test)
predicted[predicted < 0] = 0.0
```

This usually isn't a big problem, but it does make cross-validation a little trickier. Overall, I don't find this very limiting, and I love using pipelines to organize my models.

[postmortem]: http://zacstewart.com/2013/11/27/kaggle-see-click-predict-fix-postmortem.html
[sklearn]: http://scikit-learn.org/stable/
[tfidf]: http://en.wikipedia.org/wiki/Tf%E2%80%93idf
[pipeline]: http://scikit-learn.org/stable/modules/generated/sklearn.pipeline.Pipeline.html
[featureunion]: http://scikit-learn.org/stable/modules/generated/sklearn.pipeline.FeatureUnion.html
[pandas]: http://pandas.pydata.org
[langue]: http://zacstewart.com/2014/01/10/building-a-language-identifier.html
