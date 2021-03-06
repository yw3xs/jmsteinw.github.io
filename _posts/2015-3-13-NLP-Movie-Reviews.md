---
layout: post
title: Natural Language Processing in a Kaggle Competition for Movie Reviews
---

<!---
<img src='/images/Proj2_images/Movie_thtr.jpg', width = 800, height = 600>
-->
![](/images/Proj2_images/Movie_thtr.jpg){:width="800px" height="600px"}
[Source](http://imgkid.com/movie-theater-wallpaper.shtml)

I decided to try playing around with a Kaggle competition. In this case, I entered the ["When bag of words meets bags of popcorn"](http://www.kaggle.com/c/word2vec-nlp-tutorial) contest. This contest isn't for money; it is just a way to learn about various machine learning approaches. 

The competition was trying to showcase Google's [Word2Vec](https://code.google.com/p/word2vec/). This essentially uses deep learning to find features in text that can be used to help in classification tasks. Specifically, in the case of this contest, the goal involves labeling the sentiment of a movie review from IMDB. Ratings were on a 10 point scale, and any review of 7 or greater was considered a positive movie review.

Originally, I was going to try out Word2Vec and train it on unlabeled reviews, but then one of the competitors [pointed out](http://www.kaggle.com/c/word2vec-nlp-tutorial/forums/t/11261/beat-the-benchmark-with-shallow-learning-0-95-lb) that you could simply use a less complicated classifier to do this and still get a good result. 

I decided to take this basic inspiration and try a few various classifiers to see what I could come up with. The highest my score received was 6th place back in December of 2014, but then people started using [ensemble methods](http://sebastianraschka.com/Articles/2014_ensemble_classifier.html) to combine various models together and get a perfect score after a lot of fine tuning with the parameters of the ensemble weights. 

Hopefully, this post will help you understand some basic NLP (Natural Language Processing) techniques, along with some tips on using [scikit-learn](http://scikit-learn.org/stable/) to make your classification models.

## Cleaning the Reviews

The first thing we need to do is create a simple function that will clean the reviews into a format we can use. We just want the raw text, not all of the other associated HTML, symbols, or other junk. 

We will need a couple of very nice libraries for this task: BeautifulSoup for taking care of anything HTML related and re for regular expressions. 

```python
import re
from bs4 import BeautifulSoup 
```
Now set up our function. This will clean all of the reviews for us.

```python
def review_to_wordlist(review):
    '''
    Meant for converting each of the IMDB reviews into a list of words.
    '''
    # First remove the HTML.
    review_text = BeautifulSoup(review).get_text()
        
    # Use regular expressions to only include words.
    review_text = re.sub("[^a-zA-Z]"," ", review_text)
        
    # Convert words to lower case and split them into separate words.
    words = review_text.lower().split()
       
    # Return a list of words
    return(words)
```
Great! Now it is time to go ahead and load our data in. For this, pandas is definitely the library of choice. If you want to follow along with a downloaded version of the attached IPython notebook yourself, make sure you obtain the [data](http://www.kaggle.com/c/word2vec-nlp-tutorial/data) from Kaggle. You will need a Kaggle account in order to access it.

```python
import pandas as pd


train = pd.read_csv('labeledTrainData.tsv', header=0,
                delimiter="\t", quoting=3)
test = pd.read_csv('testData.tsv', header=0, delimiter="\t",
                quoting=3 )
                
# Import both the training and test data.
```

Now it is time to get the labels from the training set for our reviews. That way, we can teach our classifier which reviews are positive vs. negative.

```python
y_train = train['sentiment']
```
Now we need to clean both the train and test data to get it ready for the next part of our program. 

```python
traindata = []
for i in xrange(0,len(train['review'])):
    traindata.append(" ".join(review_to_wordlist(train['review'][i])))
    testdata = []
    for i in xrange(0,len(test['review'])):
        testdata.append(" ".join(review_to_wordlist(test['review'][i])))
```

## TF-IDF Vectorization

The next thing we are going to do is make TF-IDF (term frequency-interdocument frequency) vectors of our reviews. In case you are not familiar with what this is doing, essentially we are going to evaluate how often a certain term occurs in a review, but normalize this somewhat by how many reviews a certain term also occurs in. [Wikipedia](http://en.wikipedia.org/wiki/Tf%E2%80%93idf) has an explanation that is sufficient if you want further information. 

This can be a great technique for helping to determine which words (or ngrams of words) will make good features to classify a review as positive or negative. 

To do this, we are going to use the TFIDF vectorizer from scikit-learn. Then, decide what settings to use. The documentation for the TFIDF class is available [here](http://scikit-learn.org/stable/modules/generated/sklearn.feature_extraction.text.TfidfVectorizer.html).

In the case of the example code on Kaggle, they decided to remove all stop words, along with ngrams up to a size of two (you could use more but this will require a LOT of memory, so be careful which settings you use!)

```python
from sklearn.feature_extraction.text import TfidfVectorizer as TFIV


tfv = TFIV(min_df=3,  max_features=None, 
        strip_accents='unicode', analyzer='word',token_pattern=r'\w{1,}',
        ngram_range=(1, 2), use_idf=1,smooth_idf=1,sublinear_tf=1,
        stop_words = 'english')
```
Now that we have the vectorization object, we need to run this on all of the data (both training and testing) to make sure it is applied to both datasets. This could take some time on your computer!

```python
X_all = traindata + testdata # Combine both to fit the TFIDF vectorization.
lentrain = len(traindata)

tfv.fit(X_all) # This is the slow part!
X_all = tfv.transform(X_all)

X = X_all[:lentrain] # Separate back into training and test sets. 
X_test = X_all[lentrain:]
```

## Making Our Classifiers 

Because we are working with text data, and we just made feature vectors of every word (that isn't a stop word of course) in all of the reviews, we are going to have sparse matrices to deal with that are quite large in size. Just to show you what I mean, let's examine the shape of our training set. 

```python
X.shape
```
```python
(25000, 309798)
```


That means we have 25,000 training examples (or rows) and 309,798 features (or columns). We need something that is going to be somewhat computationally efficient given how many features we have. Using something like a random forest to classify would be unwieldy (plus random forests can't work with sparse matrices anyway yet in scikit-learn). That means we need something lightweight and fast that scales to many dimensions well. Some possible candidates are:

* Naive Bayes
* Logistic Regression
* SGD Classifier (utilizes Stochastic Gradient Descent for much faster runtime)

Let's just try all three as submissions to Kaggle and see how they perform. 

First up: Logistic Regression (see the scikit-learn documentation [here](http://scikit-learn.org/stable/modules/generated/sklearn.linear_model.LogisticRegression.html)).

While in theory L1 regularization should work well because p>>n (many more features than training examples), I actually found through a lot of testing that L2 regularization got better results. You could set up your own trials using scikit-learn's built-in GridSearch class, which makes things a lot easier to try. I found through my testing that using a parameter C of 30 got the best results.

```python
from sklearn.linear_model import LogisticRegression as LR
from sklearn.grid_search import GridSearchCV


grid_values = {'C':[30]} # Decide which settings you want for the grid search. 
    
model_LR = GridSearchCV(LR(penalty = 'L2', dual = True, random_state = 0), 
                        grid_values, scoring = 'roc_auc', cv = 20) 
# Try to set the scoring on what the contest is asking for. 
# The contest says scoring is for area under the ROC curve, so use this.
                            
model_LR.fit(X,y_train) # Fit the model.
```
```python
GridSearchCV(cv=20, estimator=LogisticRegression(C=1.0, class_weight=None, dual=True, 
             fit_intercept=True, intercept_scaling=1, penalty='L2', random_state=0, tol=0.0001),
        fit_params={}, iid=True, loss_func=None, n_jobs=1,
        param_grid={'C': [30]}, pre_dispatch='2*n_jobs', refit=True,
        score_func=None, scoring='roc_auc', verbose=0)
```


You can investigate which parameters did the best and what scores they received by looking at the model_LR object.

```python
model_LR.grid_scores_
```


```python
[mean: 0.96459, std: 0.00489, params: {'C': 30}]
```


```python
model_LR.best_estimator_
```


```python
LogisticRegression(C=30, class_weight=None, dual=True, fit_intercept=True,
            intercept_scaling=1, penalty='L2', random_state=0, tol=0.0001)
```


Feel free, if you have an interactive version of the notebook, to play around with various settings inside the <i>grid_values</i> object to optimize your ROC AUC score. Otherwise, let's move on to the next classifier, Naive Bayes. 

Unlike Logistic Regression, Naive Bayes doesn't have a regularization parameter to tune. You just have to choose which "flavor" of Naive Bayes to use. 

According to the [documentation on Naive Bayes from scikit-learn](http://scikit-learn.org/0.13/modules/naive_bayes.html), Multinomial is our best version to use, since we no longer have just a 1 or 0 for a word feature: it has been normalized by TF-IDF, so our values will be BETWEEN 0 and 1 (most of the time, although having a few TF-IDF scores exceed 1 is technically possible). If we were just looking at word occurrence vectors (with no counting), Bernoulli would have been a better fit since it is based on binary values. 

Let's make our Multinomial Naive Bayes object, and train it.

```python
from sklearn.naive_bayes import MultinomialNB as MNB


model_NB = MNB()
model_NB.fit(X, y_train)
```


```python
MultinomialNB(alpha=1.0, class_prior=None, fit_prior=True)
```


Pretty fast, right? This speed comes at a price, however. Naive Bayes assumes all of your features are ENTIRELY independent from each other. In the case of word vectors, that seems like a somewhat reasonable assumption but with the ngrams we included that probably isn't always the case. Because of this, Naive Bayes tends to be less accurate than other classification algorithms, especially if you have a smaller number of training examples. 

Why don't we see how Naive Bayes does (at least in a 20 fold CV comparison) so we have a rough idea of how well it performs compared to our Logistic Regression classifier?

You could use GridSearch again, but that seems like overkill. There is a simpler method we can import from scikit-learn for this task.

```python
from sklearn.cross_validation import cross_val_score
import numpy as np


print "20 Fold CV Score for Multinomial Naive Bayes: ", np.mean(cross_val_score
                                                                    (model_NB, X, y_train, cv=20, scoring='roc_auc'))
# This will give us a 20-fold cross validation score that looks at ROC_AUC so we can compare with Logistic Regression. 
```
```python
20 Fold CV Score for Multinomial Naive Bayes:  0.949631232
```

Well, it wasn't quite as good as our well-tuned Logistic Regression classifier, but that is a pretty good score considering how little we had to do!

One last classifier to try is the [SGD classifier](http://scikit-learn.org/stable/modules/generated/sklearn.linear_model.SGDClassifier.html), which comes in handy when you need speed on a really large number of training examples/features. 

Which machine learning algorithm it ends up using depends on what you set for the loss function. If we chose loss = 'log', it would essentially be identical to our previous logistic regression model. We want to try something different, but we also want a loss option that includes probabilities. We need those probabilities if we are going to be able to calculate the area under a ROC curve. Looking at the documentation, it seems a 'modified_huber' loss would do the trick! This will be a Support Vector Machine that uses a linear kernel. 



```python
from sklearn.linear_model import SGDClassifier as SGD


sgd_params = {'alpha': [0.00006, 0.00007, 0.00008, 0.0001, 0.0005]} # Regularization parameter
    
model_SGD = GridSearchCV(SGD(random_state = 0, shuffle = True, loss = 'modified_huber'), sgd_params, scoring = 'roc_auc', cv = 20) # Find out which regularization parameter works the best. 
                            
model_SGD.fit(X, y_train) # Fit the model.
```


```python
GridSearchCV(cv=20, estimator=SGDClassifier(alpha=0.0001, class_weight=None, epsilon=0.1, eta0=0.0, fit_intercept=True, l1_ratio=0.15, learning_rate='optimal',
           loss='modified_huber', n_iter=5, n_jobs=1, penalty='l2',
           power_t=0.5, random_state=0, shuffle=True, verbose=0,
           warm_start=False),
           fit_params={}, iid=True, loss_func=None, n_jobs=1,
           param_grid={'alpha': [6e-05, 7e-05, 8e-05, 0.0001, 0.0005]},
           pre_dispatch='2*n_jobs', refit=True, score_func=None,
           scoring='roc_auc', verbose=0)
```


Again, similar to the Logistic Regression model, we can see which parameter did the best.

```python
model_SGD.grid_scores_
```


```python
[mean: 0.96477, std: 0.00484, params: {'alpha': 6e-05},
mean: 0.96484, std: 0.00481, params: {'alpha': 7e-05},
mean: 0.96486, std: 0.00480, params: {'alpha': 8e-05},
mean: 0.96479, std: 0.00480, params: {'alpha': 0.0001},
mean: 0.95869, std: 0.00484, params: {'alpha': 0.0005}]
```


Looks like this beat our previous Logistic Regression model by a very small amount. Now that we have our three models, we can work on submitting our final scores in the proper format. It was found that submitting predicted probabilities of each score instead of the final predicted score worked better for evaluation from the contest participants, so we want to output this instead. 

First, do our Logistic Regression submission. 



```python
LR_result = model_LR.predict_proba(X_test)[:,1] # We only need the probabilities that the movie review was a 7 or greater. 
LR_output = pd.DataFrame(data={"id":test["id"], "sentiment":LR_result}) # Create our dataframe that will be written.
LR_output.to_csv('Logistic_Reg_Proj2.csv', index=False, quoting=3) # Get the .csv file we will submit to Kaggle.
```
Repeat this with the other two.

```python
# Repeat this for Multinomial Naive Bayes

MNB_result = model_NB.predict_proba(X_test)[:,1]
MNB_output = pd.DataFrame(data={"id":test["id"], "sentiment":MNB_result})
MNB_output.to_csv('MNB_Proj2.csv', index = False, quoting = 3)
    
# Last, do the Stochastic Gradient Descent model with modified Huber loss.
    
SGD_result = model_SGD.predict_proba(X_test)[:,1]
SGD_output = pd.DataFrame(data={"id":test["id"], "sentiment":SGD_result})
SGD_output.to_csv('SGD_Proj2.csv', index = False, quoting = 3)
```
Submitting the SGD result (using the linear SVM with modified Huber loss), I received a score of 0.95673 on the Kaggle [leaderboard](http://www.kaggle.com/c/word2vec-nlp-tutorial/leaderboard). That was good enough for sixth place back in December of 2014. 

## Ideas for Improvement and Summary

In this post, we examined a text classification problem and cleaned unstructured review data. Next, we created a vector of features using TF-IDF normalization on a Bag of Words. We then trained these features on three different classifiers, some of which were optimized using 20-fold cross-validation, and made a submission to a Kaggle competition.

Possible ideas for improvement:

- Try increasing the number of ngrams to 3 or 4 in the TF-IDF vectorization and see if this makes a difference
- Blend the models together into an ensemble that uses a majority vote for the classifiers
- Try utilizing Word2Vec and creating feature vectors from the unlabeled training data. More data usually helps!

If you would like the IPython Notebook for this blog post, you can find it [here.](http://nbviewer.ipython.org/github/jmsteinw/Notebooks/blob/master/NLP_Movies.ipynb)
