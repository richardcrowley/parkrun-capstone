# Data Science Capstone Project - Predicting parkrun results

This project was completed as part of the Data Science Immersive course at General Assembly in December 2020.
Note that the data is not included in this repo but notebooks with code used to acquire, clean, analyse and model the data are available.

## Background

[Parkrun](https://www.parkrun.org.uk/) organise free weekly timed runs, delivered by local volunteers, in 20 different countries globally.  People of all abilities are encouraged to run, walk or jog 5km at their local course with events taking place every Saturday morning.  Results are published online shortly afterwards.  Parkrun started in 2004 and now has 672 locations in the UK alone.  To date, over 2.3 million unique participants have crossed the finish line a total of 34.8 million times.  As a keen parkrunner I saw this as a great dataset to explore.

## Goals

Volunteering at parkrun is great fun but most regulars would agree that the job of timekeeper is the most stressful.  The most popular events can have over 2000 participants on a busy day, with most crossing the finishing line in a 45 minute window.  Even with experienced volunteers who are speedy on the stopwatch buttons, sometimes things can, and do, go wrong.

**Can we train a model to predict finish times and replace the stopwatch?**


1. [Acquire data](#Acquire-Data) - data will be scraped and parsed using Requests and Beautiful Soup
2. [Data Cleaning](#Data-Cleaning) - identify missing or inconsistent data.  Store in a clean format ready for processing
3. [EDA](#EDA) - explore the data to identify trends and correlations
4. [Modelling](#Modelling) - train a range of regression models, score, cross-validate and test on unseen data
5. [Evaluation](#Evaluation) - Evaluate and assess limitations of best model in practice against stated aim
6. [Conclusions](#Conclusions)

## Acquire Data

I used the Python libraries Requests and Beautiful Soup to scrape and parse results data published online.  Initially I had hoped to collect data from all past events across the UK but soon found that it would be sufficient and more practical to limit the number of events and timescale considered.  When Parkrun started in 2004 there was one event with 13 participants - a great start but certainly an outlier when considering the scale of current events.  I decided to focus on the Greater London area as it has a high density of events, with many runners participating at multiple locations over time, known as Parkrun Tourism!  I hoped that this data would be useful in identifying characteristics of different events and might allow other approaches including clustering and network analysis of running communities at a later stage.

Data acquired:
* 52 event locations in the Greater London Area
* 4 years' event history (14/05/2016 - 14/03/2020, up to 205 events)
* 269,697 unique participants
* 2,694,007 finish time records

The code used for web scraping is included in this notebook - [London Data Collector](london_data_collector.ipynb)

This map shows the locations of the events in Greater London as well as the total number of finishes in my dataset.
![Event locations and number of finishers](images/event_map_4.png)


## Data Cleaning

Given the size of the dataset I removed repeated text to reduce filesize, storing athlete and event data in separate reference files, linked by Athlete ID and Event Index respectively.  This left the following columns:

| Column | Description |
| --- | ----------- |
| event_index | event location |
| event_no | numbered instance of the event |
| date | date of the event |
| positions | finish position |
| athlete_no | unique participant ID |
| total_parkruns | no of times the finisher has participated in a parkrun |
| run_time | finish time (seconds) |
| event_PB | previous best time at the same event for participant |
| in_club | the participant is registered with a running club (1) or not (0) |
| age_groups | gender and age group category of the participant |
| age_grades | score measuring performance against peers in age group |


For more information on Age Grades see - [Parkrun Age Grades](https://support.parkrun.com/hc/en-us/articles/200565263-What-is-age-grading-)


Data cleaning was, as always, a fairly long and iterative process so I will just highlight a few of the challenges that it presented.

* Unknown Runners

Not all participants choose to register with parkrun so that their results are recorded and published online.  These appear in the results tables as "Unknown Runner", occupying a finish position but without any further data, including finish time.  There were a total of 254,545 unknown runners in the dataset.  

* Wheelchair participants

There were 137 finishes by wheelchair participants in the dataset.  The age groups for these participants are formed in a different way and course conditions will have a different impact on times so these were dropped for the purpose of modelling.

* No Timekeeper events

There were two event instances where there was no volunteer timekeeper so all finishes were registered with the same time of 59:59.  These were dropped.


## EDA

Next I performed thorough EDA to get to know the data, the distributions of the features and the correlations between them.
I have summarised the key findings and plots in the following notebook:
[EDA_parkrun](EDA_parkrun.ipynb)

### The target

I will be predicting finish times so the target variable is **run_time**

The distribution of run_time is heavily right skewed with a long tail of slower times.  Therefore a log transform was applied as a pre-processing step, bringing it closer to a normal distribution.

### Features

I derived four further features from the data to attempt to gain more information on how the conditions on a particular day affect finish times.

| Feature | Description | Measures |
| ---- | ---- | ---- |
| count_finishers | no. of finishers at that event on that date | How busy it is - generally busier days result in slower times |
| med_course_time | the median run_time for all past dates at this event | How difficult the course is |
| month | month of the year | Relates to the type of participants (e.g. new runners in January, kids in school holidays) |
| season | winter (Jan-Mar), spring (Apr-Jun), summer (Jul-Sep), autumn (Oct-Dec) | The conditions underfoot |

Information on course difficulty (flat/hilly, on/off road etc.) is contained in the dataset but a more definitive and detailed rating would have been ideal.  This would have to be manually collected.

Conditions - I looked in to acquiring weather data for the each event and date. However the weather on the day has a much lower impact on runners than the conditions underfoot which are a product of cumulative weather over a period of time.  I chose to use season as a crude indicator of this instead, but again a manual categorisation of conditions could be performed restrospectively if time allowed.

Another pre-processing step was to take log transform of the event_PB and positions features, as these both had a similar heavy skew as the target variable, run_time.


## Modelling

### Preparing the data

* Feature selection

I selected features that contained information that would only be available in the absence of stopwatch timing, dropping age grade scores which are calculated from finish times.

regression_cols = ['event_index', 'positions',
       'total_parkruns', 'event_PB', 'in_club', 'age_groups',
       'month', 'count_finishers', 'season',
       'med_course_time']

target_col = ['run_time']

* Transformations

I applied log transfors to positions, event_PB and run_time.

* Train-Test Split

I applied an 80:20 train-test split to the data.

* Standardisation

I applied the StandardScaler to the features as many of the models I will test require this.

### Fitting and scoring models

I fitted a range of regression models on the training set and obtained R2 scores for training and test set.  I performed 5-fold cross validation and obtained mean CV score to check the generalisability of the model, and also calculated MSE, RMSE and MAE of the predictions.

Finding the best model - Scores

![Regression Model scoring](images/scores_1.png)

The simple Linear Regression model performs quite well with consistent train, test and mean CV scores.  Adding regularisation with the Ridge and Elastic Net models did not alter the results. The Decision Tree Regressor model showed significant improvement in R2 scores and errors, so I tested other tree-based models for comparison.

<details>
    
<summary> View Decision Tree graph visualisation </summary>

![Decision Tree graph](images/decision_tree.png)

</details>

The Gradient Boosting Regressor performed similarly to the Decision Tree but the training time was too long to allow for a full gridsearch of hyperparmeters (approx 6 hours to fit and cross-validate).  Therefore a Histogram-based Gradient Boosting regressor was used, which offers significantly faster performance (approx 30 mins to fit and cross-validate) and this produced the best results.

![Tree based model scores](images/scores_2.png)


## Evaluation

Examples of the best models and evaluation can be found in the following notebook:
[Modelling_parkrun](Modelling_parkrun.ipynb)

Included in this notebook:
* Decision Tree Regressor
* Histogram-based Gradient Boosting Regressor
* Evaluation of predictions for two individual events

### Limitations

**Has the goal of predicting finish times with sufficient accuracy to replace the stopwatch been achieved?**

Whilst overall the model scores very well on the full dataset, when applying to single event to predict times it has some significant limitations

* Different conditions are not well captured

This is clearly illustrated in the evaluation of the Wimbledon Common event, where conditions were particulary muddy and difficult resulting in the model predicting significantly faster times than were observed. We can see from the feature importances for the best models that the event_index, season and month features, which do contain some information on the expected conditions have low importance so this is to be expected.

* Predicted times do not increase monotonically with position

This is a major limitation as it means the results do not make sense in practice and could not be used without adjustment.  It would be difficult to improve much further on this as the predictions are largely based on previous times, but in reality this is a fun, amateur event and participants do not perform consistently in relation to their best time.  They may sometimes run slower to take part with friends, or run with dogs or even buggies, which brings an unpredictable variation in times.

* Unknown Runners and new participants

Additional processing of results would need to be developed for Unknown Runners, who do not register for their time, and new participants, who do not have previous statistics, as the model relies on the these features to make predicitons.

## Conclusions

Overall this has been an interesting and worthwhile project.  A large amount of data was collected through web scraping with nearly 2.7 million observations of finish times across 52 event locations in Greater London.  A full range of regression modelling techniques were employed and the models scored very highly on the full dataset.  However evaluating the model against the stated aim of predicting all finish times for an individual event revealed some significant limitations, particularly in unusual conditions.  The next steps for this project would be:
- Add more detailed features to capture unique event conditions, which would likely involve a manual approach in assessing individual events.
- Collect data on a wider range of events, retrain and evaluate the model.
- Use other techniques such as clustering and network analysis to identify event and runner characteristics and communities.

If you found this project interesting and would like to discuss further then please feel free to contact me via Github or [LinkedIn](https://www.linkedin.com/in/richard-crowley-10/).


