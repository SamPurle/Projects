---
layout: post
title:  "Titanic - A Machine Learning Classification model to predict survival"
date:   2020-06-19 12:53:00 +0100
categories: project upload
---

### Overview

This project aims to make use of a Random Forest Classifier model to predict whether passengers survived the Titanic disaster, based on a variety of factors. Although somewhat morbid, this provides a good introduction to Machine Learning model construction, and key concepts such as Cleaning and Encoding.

### Data Structure

The raw data is available in two separate '.csv' files: for Train and Test data. The training data contains the "ground truth": whether the passenger survived the disaster or not, represented as a binary value. The other columns are:

- **Passenger Class:** The class of ticket held by the passenger, which can be used as a proxy for Socio-Economic Status
- **Name:** The passenger's Name
- **Sex:** The passenger's biological Sex
- **Age:** The Age of the passenger
- **Siblings and Spouses ("SibSp"):** The number of Siblings and Spouses of the passenger that were also on-board
- **Parents and Children ("Parch"):** The number of Parents and Children of the passenger that were also on-board
- **Ticket:** The passenger's ticket number
- **Fare:** The fare paid by the passenger to obtain their Ticket
- **Cabin:** The passenger's Cabin Number
- **Embarked:** The port of Embarkation

A sample of this data is shown below:

<table>
  {% for row in site.data.dfTrainHead %}
    {% if forloop.first %}
    <tr>
      {% for pair in row %}
        <th>{{ pair[0] }}</th>
      {% endfor %}
    </tr>
    {% endif %}

    {% tablerow pair in row %}
      {{ pair[1] }}
    {% endtablerow %}
  {% endfor %}
</table>

### Cleaning

The first stage during data analysis is to "clean" the data, and ensure that it exists in a usable format. For this purpose I find it useful to separate the features used for prediction from the Ground Truth, and concatenate the two DataFrames. This allows cleaning operations to be performed on the two datasets simultaneously, and prevents issues arriving during encoding when a categorical value appears in one dataset but not the other. It is important to maintain the ability to separate the two datasets after Cleaning and Feature Engineering to avoid cross-contamination. In this case it is possible to simply record the "PassengerId" index for each original DataFrame.

Next, it is necessary to identify which columns contain sufficient information to be of use during model construction. The Null values in each column can be counted and divided by the length of the DataFrame, which will show the percentage of values in each column containing Nulls. For this model, any column containing 25% or more Null values was dropped from the dataset.

<table>
  {% for row in site.data.Nulls %}
    {% if forloop.first %}
    <tr>
      {% for pair in row %}
        <th>{{ pair[0] }}</th>
      {% endfor %}
    </tr>
    {% endif %}

    {% tablerow pair in row %}
      {{ pair[1] }}
    {% endtablerow %}
  {% endfor %}
</table>

The only column which satisfies this condition is the "Cabin" column, which contains 77% Nulls. This is far too high a percentage for Imputing to be appropriate or effective, and so this data cannot be included in the model. This leaves 3 columns remaining for which Imputing is necessary:

- Age
- Embarked
- Fare

### Imputing

Prior to encoding and modelling the data, it is necessary to specify how null values are to be handled. Although this can be done using the SimpleImputer object from the Scikit-Learn library, I found that better results were achieved through pandas' "fillna" method to calculate continuous values. The Embarkation column was imputed using the SimpleImputer object with the "most_frequent" strategy.

{% highlight python %}

"""
CatImpute: A function to impute nulls
"""

def CatImpute (df, NULL_THRESH):

    # Drop high-null columns

    NullPer = df.isnull().sum() / len(df)
    NullPer.sort_values(ascending = False, inplace = True)        
    NullCols = NullPer.loc[NullPer > NULL_THRESH].index      
    df.drop(columns = NullCols, inplace = True)

    # Impute low-null columns

    df['Age'].fillna(df.groupby(['Sex','Pclass'])['Age'].transform('mean'), inplace = True)
    df['Fare'].fillna(df.groupby(['Sex','Pclass'])['Fare'].transform('mean'), inplace = True)
    df['Embarked'] = SimpleImputer(strategy = 'most_frequent').fit_transform(df[['Embarked']])

    return df

{% endhighlight %}

### Extraction

In some cases, it may be necessary to extract general information from more specific data within the dataset. In this example, I decided it was appropriate to extract a passenger's title from their Name. The exact Name of a passenger is unlikely to be of any use, whereas their Title can be used as a proxy for Socio-Economic Status, Age, Marital Status and Profession.

This was a relatively simple task which necessitated the use of string operations. This was made easy due to the fact that the naming conventions were entirely uniform throughout the dataset, and would have been much more complex had the Passenger Names been recorded differently (potentially involving string indexing from a Dictionary of common titles across different countries and cultures).

{% highlight python %}

"""
Title: A function to extract title from Passenger Name
"""

def Title(df, FANCY_THRESH):

    # Extract title from name

    df['Title'] = df['Name'].str.split(', ', expand = True)[1].str.split('.', expand = True)[0]
    df.drop(columns = 'Name', inplace = True)

    # Group rare titles into one category

    TitleCounts = df.groupby(['Sex', 'Title']).Title.count()

    FancyTitles = TitleCounts.loc[TitleCounts.values < 10]

    MaleFancy = FancyTitles.loc['male']
    MaleIndexReplace = df.join(MaleFancy, on = 'Title', how = 'inner', rsuffix = 'F').index
    df.loc[MaleIndexReplace, 'Title'] = 'FancyMan'

    FemaleFancy = FancyTitles.loc['female']
    FemaleIndexReplace = df.join(FemaleFancy, on = 'Title', how = 'inner', rsuffix = 'F').index
    df.loc[FemaleIndexReplace, 'Title'] = 'FancyLady'

    return df

{% endhighlight %}

There were 4 common titles within the Dataset: "Mr" and "Master" for males and "Mrs" and "Miss" for females. There were many uncommon titles within the Dataset, such as "Jonkheer" and "The Countess" - all of which implied high Socio-Economic Status and none of which pertained to 10 passengers or more. As such, these titles were grouped together as "FancyMan" for males and "FancyLady" for females.

### Visualisation

It is often useful to visualise data for the purposes of Exploratory Data Analysis, and to identify general trends which could be further investigated and modelled. The Seaborn library has a very useful "heatmap" function which can help to identify correlations within the dataset as a whole.

{% highlight python %}

"""
Correlate: A function to visualise correlations within the dataset
"""

def Correlate(dfTrain):

    dfTrain['Sex'].replace({'male' : 0, 'female' : 1}, inplace = True)
    Corr = dfTrain.corr()    
    CorrPlot = sns.heatmap(Corr, cmap = 'RdYlGn', square = True, linewidth = 0.5, center = 0)
    plt.title('Plot showing correlations within the data')

    plt.savefig('{}/CorrPlot.png'.format(ASSET_FILEPATH))

    return

{% endhighlight %}

{:refdef: style="text-align: center;"}
<img src="{{site.url}}/{{site.baseurl}}/assets/Titanic/CorrPlot.png">
{: refdef}

Although this plot does not show detailed information about the data, it can be used to highlight areas for investigate. The two features that appear to have the strongest influence on a passenger's Survival are Passenger Class and Sex - with Females having a higher likelihood of Survival than Males.

Plots can be generated on a feature-by-feature basis to investigate trends in more detail. It is possible to generate these visualisations iteratively as shown by the function below, although I have generated some plots manually to provide cleaner formatting and better legibility.

{% highlight python %}

"""
Visualise: A function to visualise the data to observe trends
"""

def Visualise(df, CONT_COLS):

    Cols = list(df.columns)
    DisCols = [x for x in Cols if x not in CONT_COLS]

    for xCol in DisCols:

        sns.barplot(x = df[xCol], y = yTrain)
        plt.title('Plot showing how {} impacted Survival'.format(xCol))
        plt.savefig('{}/{}_Plot.png'.format(ASSET_FILEPATH, xCol))
        plt.show()              

    return DisCols

{% endhighlight %}

#### Sex and Passenger Class

{:refdef: style="text-align: center;"}
<img src="{{site.url}}/{{site.baseurl}}/assets/Titanic/SexPlot.png">
{: refdef}

As expected, there was a vast difference in survival rates between Males and Females. Virtually all Women travelling in 1st and 2nd class survived the disaster, and even Women in 3rd class had an approximately even chance of surviving the event.

The situation for Males was drastically different. Men travelling in First Class had a ~35% chance of survival, with this figure being closer to 10% for those travelling in 2nd and 3rd class.

It is interesting that there was a large drop-off in survival chance between 2nd and 3rd class for Women, whereas this difference was most pronounced between 1st and 2nd class for Men.

#### Siblings and Spouses

{:refdef: style="text-align: center;"}
<img src="{{site.url}}/{{site.baseurl}}/assets/Titanic/SibPlot.png">
{: refdef}

Single Adults and only Children had a relatively poor chance of survival, at approximately 35%. Passengers travelling with just 1 other sibling or a partner had a much greater chance or Survival at ~55%, which decreased with additional Siblings and Spouses.

#### Port

{:refdef: style="text-align: center;"}
<img src="{{site.url}}/{{site.baseurl}}/assets/Titanic/PortPlot.png">
{: refdef}

Passengers embarking at Southampton and Queenstown had an approximately equal chance of surviving at around 35%. Those who boarded at Cherbourg had a much higher 55% chance of survival. Unfortunately it is not possible to determine whether those boarding at Cherbourg were generally accommodated within different decks of the ship with raised their chances of survival. It is possible, however, to determine whether passengers boarding at Cherbourg had different Passenger Class and Sex distributions compared to those who boarded at the other ports.

{:refdef: style="text-align: center;"}
<img src="{{site.url}}/{{site.baseurl}}/assets/Titanic/CherPlot.png">
{: refdef}

As can be seen in the plot above, passengers embarking at Cherbourg were much more likely to be travelling in 1st Class and much less likely to be travelling in 3rd Class.

{:refdef: style="text-align: center;"}
<img src="{{site.url}}/{{site.baseurl}}/assets/Titanic/CherSex.png">
{: refdef}

Variations in the Sex distribution between different ports is likely insignificant, and so this trend can likely be explained primarily as a function of Passenger Class.

#### Title

{:refdef: style="text-align: center;"}
<img src="{{site.url}}/{{site.baseurl}}/assets/Titanic/TitlePlot.png">
{: refdef}

Almost all Women with rare Titles survived the disaster - having a survival chance exceeding that of unmarried Women and Girls under the "Miss" title. This trend was not the same for Males: Men with rare Titles had a ~35% chance of survival whereas boys denoted by the "Master" Title had a 55% chance of surviving.

### Encoding

After null values within the dataset have been handled, it is necessary to encode the categorical features such that they can be processed by Machine Learning algorithms. In the cases of ordinal data, this can be done in place as the different categories are in a defined order. Passenger Class is an example of ordinal data: as this feature can only take on three different values which correlate to the passenger's Socio-Economic Status. Sex is generally not considered to be an ordinal variable, but in this example could be encoded in-place as Females had a substantially higher chance of survival.

{% highlight python %}

"""
Encode: A function to encode categorical variables
"""

def Encode(df, CAT_NUM):

    CatCols = list(df.columns.drop(df.get_numeric_data().columns))
    CatCols.extend(CAT_NUM)

    Enc = OneHotEncoder(sparse = False)

    for x in CatCols:
        EncCol = Enc.fit_transform(df[[x]])
        EncNames = Enc.get_feature_names([x])
        EncDf = pd.DataFrame(EncCol)
        EncDf.columns = EncNames
        EncDf.index = df.index

        df = df.join(EncDf)    
        df.drop(columns = x, inplace = True)

    return df

{% endhighlight %}

### Feature Engineering

It is often beneficial to extract additional meaning from existing data to pass to the Machine Learning model. This was somewhat difficult for this dataset, as the number of measured features is very low. Two variables that can be inferred from others include the passenger's Family Size, and whether they were Alone or not.

{% highlight python %}

"""
Engineer: Initial, basic feature Engineering to extract additional meaning from other columns.
"""

def Engineer(df):

    df['FamSize'] = df['Parch'] + df['SibSp']

    df['Alone'] = 0
    df.loc[df['FamSize'] == 0, 'Alone'] = 1

    return df

df = Engineer(df)

{% endhighlight %}

Another feature that I considered extracting was the number of passengers in each Cabin. However, as discussed above this feature too incomplete to be able to extract meaningful information from it.

### Modelling

#### Decision Trees

Decision Trees are generally considered to be the simplest Machine Learning algorithm. They function by applying a Boolean test to the features within a test to cause a "split" in the data, which results in informational gain and a reduction in impurity. These splits continue to be made based upon Boolean tests on all features within a dataset, until a "leaf node" is reached.

During predictions these Boolean tests are applied to each sample within the test data, until a leaf node is reached. The modal target value of training samples within this node will determine the prediction that is made for the test sample.

These trees are very easy to construct with the Scikit-Learn library:

{% highlight python %}

"""
DecTree: A function to construct a simple Decision Tree Classifier
"""

def DecTree(xTrain, yTrain):

    DecTree = DecisionTreeClassifier()
    DecTree.fit(xTrain, yTrain)

    return DecTree

{% endhighlight %}

Even a simple model such as this, with no additional optimisation achieved an accuracy of 70.8% on unseen test data. The first 2 decision layers of this tree are visualised below:

{:refdef: style="text-align: center;"}
<img src="{{site.url}}/{{site.baseurl}}/assets/Titanic/DecTree_render.png">
{: refdef}

#### Random Forests

I elected to use a Random Forest Classifier for this model, which is essentially a semi-randomised collection of Decision Trees which each "vote" on the outcome based on the supplied features. These votes are then tallied, and the most common output is used as the model's prediction.

There are a variety of hyper-parameters which can be specified during Random Forest generation. A hyper-paramater is a parameter or variable that defines how the model is trained and constructed: and therefore must be supplied by the user prior to model construction. These include:

- **n_estimators:** The number of decision trees to be used in the Forest
- **criterion:** The criterion to be used in determining logical splits within the dataset
- **max_depth:** The maximum depth, or number of splits that can be made during classification
- **min_samples_leaf:** The minimum number of samples that can be used to constitute a leaf node within a tree. This can be useful to prevent over-fitting
- **max_features:** The maximum number of features to consider when splitting the data

Random Forests are equally trivial to construct within the Scikit-Learn library:

{% highlight python %}

"""
InitialModel: A function to build and fit an initial classification model
"""

def InitialModel(xTrain, yTrain):

    Model = RandomForestClassifier(n_estimators = 100, random_state = 42, n_jobs = 3)
    Model.fit(xTrain,yTrain)

    return Model

{% endhighlight %}

#### Optimisation

Optimising these parameters can be used to substantially increase model performance. A common technique used for optimisation is the use of cross-validation: where the training data is split into a number of different "folds":

- The training data is randomly split into k number of folds
- Each fold is used as training data k-1 times
- Each fold is used as validation data 1 time, and is used to cross-validate the model

This allows for very efficient use of labelled training data, as scoring metrics can be calculated within the training data and used to evaluate the hyper-parameter selection. A particularly useful tool for hyper-parameter optimisation is the "RandomizedSearchCV" function within sklearn. This function takes in an estimator model and a specified range of hyper-parameters, and scores random combinations of these hyper-parameters to determine which give the best model performance:

{% highlight python %}

"""
Optimise: A function to optimise the model using a Random Grid Search
"""

def Optimise(xTrain, yTrain, n_iter, FOLDS):

    # Specify base model

    rfc = RandomForestClassifier(random_state = 42)
    rfc.fit(xTrain, yTrain)

    # Specify parameters to vary

    EstimatorOptions = range(1,501)
    DepthOptions = range(1,51)
    FeatureOptions = range(1,25)
    ParamGrid = dict(n_estimators = EstimatorOptions, max_depth = DepthOptions, max_features = FeatureOptions)

    # Optimise

    CVGrid = RandomizedSearchCV(rfc, ParamGrid, n_iter = n_iter, n_jobs = 3,
                       scoring = 'accuracy', cv = FOLDS, random_state = 42,
                       return_train_score = True)
    CVGrid.fit(xTrain,yTrain)

    BestEstimator = CVGrid.best_estimator_

    return BestEstimator

{% endhighlight %}

- **n_iter:** Specifies the number of times this process is iterated through, which is equal to the number of different hyper-parameter combinations that are tested
- **n_jobs:** Specifies the number of CPU threads allocated to this process
- **cv:** Specifies the number of folds to be used for cross-validation
- **random_state:** Seeds the random number generator for parameter selection

It is important to note that a "grid search" can also be performed for model optimisation. This involves supplying discrete lists of hyper-parameters to the model, which will fit and evaluate every combination thereof. This is very computationally intensive if it is desirable to investigate large ranges of hyper-parameters, although it is more likely to find local optima (as the optimiser will search every possible combination instead of random increments within a range).

After optimisation, the best Random Forest Classifier model gave an accuracy of 79.4% on unseen test data.

### Final Thoughts

I think this project provides an excellent starting point for those interested in Machine Learning. It encourages you to become familiar with core concepts, and allows you to gain an understanding of how relatively simple Machine Learning models function. Additionally, it highlights the importance of becoming familiar with the underlying dataset and identifying the merits of modelling/further research through visualisation.

Furthermore, this project demonstrates the importance of the quantity of samples and features. After researching a wide variety of notebooks on [Kaggle](https://www.kaggle.com/c/titanic/notebooks), I could not find a Machine Learning model that exceeded 83% test accuracy.
