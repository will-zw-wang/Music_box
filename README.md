# Music_Box

<img src="https://github.com/will-zw-wang/Music_box-Churn_Prediction_and_Recommender_System/blob/master/images/music_box_image.jpg" width="960" height="240">

- [**Report**](https://github.com/will-zw-wang/Music_box-Churn_Prediction_and_Recommender_System/blob/master/Music_Box_Project_Report.pdf)
- [**Code**](https://github.com/will-zw-wang/Music_box-Churn_Prediction_and_Recommender_System/tree/master/code)

## Project Objectives

X music box is a well-known music player platform and interested in **Churn Prediction and Recommendation**. To help explore this question, they have provided log data containing billions of song play, search, and download records generated by 600K users. 

## Dataset description
- Play log data 
  - 'uid', 'device', 'song_id', 'song_type', 'song_name', 'singer', 'play_time', 'song_length', 'paid_flag', 'date'
- Download log data
  - 'uid', 'device', 'song_id', 'song_name', 'singer', 'paid_flag', 'date'
- Search log data
  - 'uid', 'device', 'time_stamp', 'search_query', 'date'
- [**Raw data samples**](https://github.com/will-zw-wang/Music_box-Churn_Prediction_and_Recommender_System/tree/master/raw_log_data_samples)


## Analysis Structure
1. Data Exploration and Down Sampling by User
2. Data Processing and Feature Engineering with Spark
3. Churn Prediction Model Fitting, Models Comparison and HyperParameter Tuning
4. Churn Prediction Analysis, Insights and Next Step
5. Recommender System Model Fitting and Models Comparison
6. Recommendation Results Analysis, Insights and Next Step


## Analysis Details

### 1. Data Exploration and Down Sampling by User
- Get user id list and count  
- Remove bots and outliers
  - There were a few ‘uid’ have huge amount of records up to 10^5, we removed these bots and outliers extremely large ‘count’ larger than 99.9 percentile of all ‘count’ value.
- Apply downsampling on uid level 
  - As we had nearly 1.5 billion records generated by 600K users after moving the bots and outliers, we performed downsampling on ‘uid’ level to reduce computing pressure. 
  - Down_sample_ratio was 0.1 and finally we had records generated by 60K users with 12 million log records. 
  - 12 million log records including:
    - 0.6 million 'D' log records, ‘D’ for ‘download’.
    - 0.8 million 'S' log records, ‘S’ for ‘search’.
    - 11.0 million 'P' log records, ‘P’ for ‘play’.
- Create event table for feature generation  
- [**Detailed Code**](https://github.com/will-zw-wang/Music_box-Churn_Prediction_and_Recommender_System/blob/master/code/3_down_sampling_by_user.ipynb)

### 2. Data Processing and Feature Engineering with Spark
- Label definition
  - We defined churn and time windows as below:
    - 1 for churn: user had P/D/S entries in 'Feature window' while had no P/D/S entries in 'Label window'
    - 0 for not churn: user had P/D/S entries in both two windows
    - Label window: 2017-04-29 ~ 2017-05-12 days: 14
    - Feature window: 2017-03-30 ~ 2017-04-28 days: 30
    - **Note**: here we ignored user who had P/D/S entries in 'Label window' while had no P/D/S entries in 'Feature window'.
  - we have 36,272 churn label and 22,160 active label, two classes were comparable.
- Data cleaning
  - Data processing of "song_length" 
  - Data cleaning for "play_time"
- Feature generation
  - Generate Features A: Play time percentage related features
    - Generate features **play time percentage proportion feature**:
      - **time_percentage_0_to_20**, **time_percentage_20_to_40**, **time_percentage_40_to_60**, **time_percentage_60_to_80**, and **time_percentage_larger_than_80**.
      - **Note**: for each ‘uid’, the sum of five features above is 1.
      - **For example**: ‘time_percentage_0_to_20’ means 0<=percentage<20, which is ratio of (number of played songs with play time percentage less than 20%) to (total number of played songs) per ‘uid’
    - Generate features **Acceleration features of different level of play time percentage**: Ratio of count of songs played with >=80 percentage of nearest 7 days to that of nearest 30 days.
  - Generate Features B: Play time percentage related features
    - Generate features **Total play_time**: Total play time per ‘uid’.
    - Generate features **Acceleration features of total play_time**: Ratio of total play time of nearest 7 days to that of nearest 30 days. 
      - If the total play time acceleration ratio is less than 25%, means user had played less time in the last 7 days than average level of the last 30 days.
    - Generate features **Average play_time of played songs**: Average play time of songs per ‘uid’.
  - Generate Features C: Event related features
    - Generate features **events frequency in given windows**: Count of ‘P’, ‘D’ and ‘S’ in given windows per ‘uid’.
    - Generate features **Acceleration features of events**: Ratio of event frequency of nearest 7 days to that of nearest 30 days. 
      - If the total play time acceleration ratio is less than 25%, means user had had operated less frequently in the last 7 days than average level of the last 30 days.
  - Generate Features D: Recency related features
    - Generate features **Last Event Time from feature_window_end_date**: Measure the time gap between ‘the last active day’ and ‘feature_window_end_date’.
  - Generate Features E: Profile related features
    - Generate features **device_feature**: Device type per ‘uid’.
      - We found 20 users use multiple devices, we assigned the device label as device with more entries by correspond user, if user has the same entries number of 'android' and 'iphone', we assign its device label as 'iphone'.
- Form training data for prediction models
- Form rating data for recommendation system
  - We define ‘rating’ as max{‘play_score’, ‘download_score’}: 
    - ‘play_score’ are generate by ‘play_time_percentage_of_song_length’ with idea that the larger played percentage is, the more likely the user like the song, rules as below: 
      - 0.8 <= ‘play_time_percentage_of_song_length’, assign ‘play_score’ 5
      - 0.6 <= ‘play_time_percentage_of_song_length’ < 0.8, assign ‘play_score’ 4
      - 0.4 <= ‘play_time_percentage_of_song_length’ < 0.6, assign ‘play_score’ 3
      - 0.2 <= ‘play_time_percentage_of_song_length’ < 0.4, assign ‘play_score’ 2
      - 0 <= ‘play_time_percentage_of_song_length’ < 0.2, assign ‘play_score’ 1
      - **Note**: If per uid per song_id has multiple ratings, we take average.
    - ‘download_score’ are generate by whether user has download entry in feature window: 2017-03-30 ~ 2017-04-28 with idea that if a user downloads a song, he has great probability to like the song, rules as below: 
    - If have download entry, assign ‘download_score’ 5
    - If no download entry, assign ‘download_score’ 0
- [**Detailed Code**](https://github.com/will-zw-wang/Music_box-Churn_Prediction_and_Recommender_System/blob/master/code/4_feature_label_generation_with_spark.ipynb)

### 3. Churn Prediction Model Fitting, Models Comparison and HyperParameter Tuning
- Models comparison and reasoning
  - **Logistic Regression**
    - **AUC** of test data is **0.8866** with **Logistic Regression**.
    - We tried to improve model performance with **Random Forest**.
  - **Random Forest**
    - **AUC** of test data is **0.9061** with **Random Forest**, better than that of **Logistic Regression** with **0.8866**.
      - **Reason**: There are feature interaction and non-linearity relationship between features and target in our data set, trees algorithms can deal with these problems while logistic regression cannot.
    - We tried to further improve model performance with **Gradient Boosting Trees**.
      - **Reason**: In general, **Gradient Boosting Trees** can perform better than **Random Forest**, because it additionally tries to find optimal linear combination of trees (assume final model is the weighted sum of predictions of individual trees) in relation to given train data. This extra tuning may lead to more predictive power.
  - **Gradient Boosting Trees**
    - **AUC** of test data is **0.9036** with **Gradient Boosting Trees**, is close to that of **Random Forest** with **0.9061**.
      - **Reason**: Our **Random forest** has already performed greatly in this dataset and hard for **Gradient Boosting Trees** to perform better.
    - Thus, we chose **Random Forest** as our preferred model.
    - Then we tried HyperParameter Tuning with Grid Search for Random Forest, to figure out whether we can do better.
- **Random Forest HyperParameter Tuning with Grid Search**
  - **AUC** of test data is **0.9062** with **Random Forest HyperParameter Tuning with Grid Search**, is slightly better than that of previous **Random Forest** with **0.9061**
  - we select this model to explore the features importance to get some insights.
- [**Detailed Code**](https://github.com/will-zw-wang/Music_box-Churn_Prediction_and_Recommender_System/blob/master/code/5_Churn_Prediction_Models.ipynb) 

### 4. Churn Prediction Analysis, Insights and Next Step

- Top 10 features analysis
  - <img src="https://github.com/will-zw-wang/Music_box-Churn_Prediction_and_Recommender_System/blob/master/images/Ranked_Feature_Importance_Generated_by_Random_Forest.png"> 
  - **last_P_time_from_2017-04-28**: the larger the time gap between 'the last active day' and 'feature_window_end_date' is, the more likely the user will churn.   
  - **freq_P_last_7**: the smaller play frequency in the last 7 days, the more likely the user will churn. And its feature importance is larger than those of 'freq_P_last_14', 'freq_P_last_30', which shows user recent behavior pattern is more informative.  
  - **freq_P_last_14**: the same idea with 'freq_P_last_7'  
  - **total_play_time_7d_over_30d**: if the total play time acceleration ratio is less than 25%, means user had played less time in the last 7 days than average level of the last 30 days, the smaller the ratio is, the more likely the user will churn. 
  - **time_percentage_larger_than_80_7d_over_30d**: if the time_percentage acceleration ratio is less than 25%, means user had played less songs with high percentage in the last 7 days than average level of the last 30 days, the smaller the ratio is, the more likely the user will churn.   
  - **P_7d_over_30d**: if the play frequency acceleration ratio is less than 25%, means user had played less frequently in the last 7 days than average level of the last 30 days, the smaller the ratio is, the more likely the user will churn.   
  - **total_play_time**: the smaller total play time is, the more likely the user will churn.   
  - **freq_P_last_30**: the same idea with 'freq_P_last_7'.  
  - **freq_S_last_14**: the smaller search frequency in the last 14 days, the more likely the user will churn. Its feature importance is large perhaps because 14 days fits our label window which last 14 days better.  
  - **freq_P_last_3**: the same idea with 'freq_P_last_7', while its time window is too short to be as informative as 'freq_P_last_7'  
 
- Business model and stage analysis
  - As we are not told what business the music box is running, we will analyze different scenarios to figure out what we can do to reactive users with high churn probability:
    - 1．If it's running a freemium model: 
      - For free users with high churn probability, we can offer them one month paid version free trial to reactive them; 
      - For paid users with high churn probability, we can offer them discount or even a freemonth; 
    - 2． If it's running a paid model: 
      - For paid users with high churn probability, we can offer them discount or even a freemonth;
  - Of course, we need to consider two factors before offering free trial, discount or freemonth:
    - 1.  We should weigh the cost of doing so against the cost of acquiring another customer. 
    - 2.  We should make sure whether our music box is in the 'Stickiness' stage to improve user engagement, or in the 'Revenue' stage to improve profits.
  - **Scenarios one**:
    - If music box is in the 'Stickiness' stage, it's reasonable to offer trial, discount and freemonth even when the cost of doing so overweighs the cost of acquiring another customer to improve user engagement.
  - **Scenarios two**:
    - If music box is in 'Revenue' stage and the cost of offering trial, discount or freemonth overweighs the cost of acquiring another customer, it's better to just sending e-mail and allocate more budgets to campaigns to attract new users and improve profits.
  - We calculated the churn rate of our music box with 0.62. As the churn rate is very high, we concluded it's in the 'Stickiness' stage, thus offering free trial, discount or freemonth is reasonable, and sending e-mail which is applicable to every business model is also good choice.
  
- Insights and Recommendations
  - Users' recent behavior pattern is more informative, we should pay more attention to last 7 days and last 14 days related matric, especially frequency features and acceleration features, like 'freq_P_last_14', 'freq_P_last_7', 'P_7d_over_30d', 'total_play_time_7d_over_30d' and 'time_percentage_larger_than_80_7d_over_30d'.
  - Once the time gap between 'the last active day' and feature_window_end_date' comes to 7 days, we should pay more attention to these users; once the time gap comes to 14 days, we should take some action to reactive them, like sending e-mail to recommend songs, offering free trial, discount or freemonth.
  - Generate the churn probability of every user and rank to figure out the users most likely to churn, then send them e-mail to recommend songs, or offer them free trial, discount or freemonth to reactive them.
  - Figure out our target users who are most unlikely to churn with our model, try to figure out what’s common to this subsection of users, refocus on their needs, and grow from there. 
    - To be specific, develop features which target users care, allocate campaigns budget to markets where our target users in.

- Next step
  - Besides the insights mentioned above, I think there are aspects we can further dive deep, like:
    - If we have a hypothesis that users churn because they don't like the songs we provide, we can analyze the songs played by churn users before they churned in a suitable time windows, try to figure out what’s common to the songs driven users churn, and avoid providing these kinds.
    - It’s also important to track performance over time. If we have more data, we can see whether we’re improving or not, perform cohort analysis by comparing churn rate for each month.
    - Develop 'like' feature to build a lock-in users experience, user can 'like' the music and keep it in their personal playlist, the more songs users keep, the stickier they will be, because there’s a lot of data in place, so churn probability may be lower.

### 5. Recommender System Model Fitting and Models Comparison
- Clean data and create utility matrix
  - We build utility matrix with only users played more than five songs.
  - For the removed or new users, we can recommend popular songs at first.
- Build Recommender Systems
  - **Popularity-based recommender**
    - We defined 'popular' as songs with most played records.
    - For every new user or user played less than five songs, we build a Popularity-based recommender to recommend most popular songs at first. 
    - Our Popularity-based recommender recommended top 10 songs: 
      - [10375, 9320, 6496, 6070, 5338, 5248, 4671, 4474, 4122, 3782].
  - **Neighborhood-based Approach Collaborative Filtering Recommender：Item-Item similarity recommender**
    - For user played more than five songs, we tried Neighborhood-based approach to build an Item-Item similarity recommender here. 
    - Given a user_id and recommend 10 songs which have the largest similarities with songs the user had played before.
    - We tried to get final recommendations for a user: user_number = 100, and our Item-Item similarity recommender recommended top 10 songs: 
      - [42746, 45983, 38553, 38554, 38555, 38556, 38557, 45981, 45982, 45984]
      - With an **average absolute error** of **0. 92**.
    - Then we tried to improve performance with Matrix Factorization approach to build recommender, because matrix factorization models always perform better than neighborhood models in collaborative filtering. 
      - **Reason**: 
        - The reason is when we factorize a ‘m*n’ matrix into two ‘m*k’ and ‘k*n’ matrices we are reducing our "n"items to "k"factors, which means that instead than having our 50000 songs, we now have 500 factors where each factor is a linear combination of songs. 
        - The key is that recommending based on factors is more robust than just using song similarities, maybe a user hasn’t played the song ‘stay’ but the user might have player other songs that are related to ‘stay’ via some latent factors and those can be used.
        - The factors are called latent because they are there in our data but are not really discovered until we run the reduced rank matrix factorization, then the factors emerge and hence the "latency".
  - **Matrix Factorization Approach Collaborative Filtering Recommender：NMF**
    - The **RMSE** of **NMF** is **1.3431**, and the **average absolute error** is **0.6158**, the performance is acceptable. 
    - We tried to get final recommendations for a user: user_number = 100, and our NMF recommender recommended top 10 songs: 
      - [45979, 45977, 45978, 10569, 26479, 9938, 23038, 9869, 9880, 23062]
      - With an **average absolute error** of **0.023**.
    - The same as what we discussed above, the **average absolute error** of **NMF** for this specific user is better than **0.92** of **Item-Item similarity recommender**.
    - Then we tried TruncatedSVD to verify whether it performs better than NMF.
  - **Matrix Factorization Approach Collaborative Filtering Recommender：TruncatedSVD**
    - The **RMSE** of **TruncatedSVD** is **1.1702** and the **average absolute error** is **0.56**, which are better than scores of **NMF**(**1.3431** and **0.6158**). 
      - **Reason**: **TruncatedSVD** performs better because it has larger degree of freedom than **NMF**, to be specific, **NMF** is a specialization of **TruncatedSVD**, all values of V, W, and H in **NMF** must be non-negative.
    - Then we tried to get final recommendations for a user: user_number = 100, and our TruncatedSVD recommender recommended top 10 songs: 
      - [258, 7341, 7512, 18055, 28364, 658, 13202, 45719, 48377, 48378]
      - With an **average absolute error** of **0.044**, which is very close to **0.023** of **NMF**.
- [**Detailed Code**](https://github.com/will-zw-wang/Music_box-Churn_Prediction_and_Recommender_System/blob/master/code/6_Recommender_Systems.ipynb) 

### 6. Recommendation Results Analysis, Insights and Next Step
- Recommendation results Analysis between different recommendation systems for user: user_number = 100
  - We generated the overlap tables of the **top_10** and **top_100** results given by the four models for 'user with user_number = 100' as below:
    - <img src="https://github.com/will-zw-wang/Music_box-Churn_Prediction_and_Recommender_System/blob/master/images/The%20overlap%20of%20the%20top%2010%20recommendation%20generated%20by%20these%20four%20models.png">
    - <img src="https://github.com/will-zw-wang/Music_box-Churn_Prediction_and_Recommender_System/blob/master/images/The%20overlap%20of%20the%20top%20100%20recommendation%20generated%20by%20these%20four%20models.png">
  - From the overlap tables above, we notice that:
    - The recommended songs given by Popularity-based, Neighborhood-based approach and Matrix Factorization approach models are very different from each other, have no overlap in top 10 even in top 100 recommended songs.
    - While the recommended songs given by NMF and TruncatedSVD have no overlap in top 10 and 30% overlap in top 100 recommended songs.
- Conclusion:
  - 1. For **new user or user played less than five songs**, we can recommend most popular songs at first generated by our **Popularity-based recommender**.
  - 2. For **user played more than five songs**:
    - Given the performances of **NMF** and **TruncatedSVD** are comparable, we can have the overlap of commendation results generated by these two models as the final recommendation.
    - As the results generated by **Popularity-based**, **Neighborhood-based Approach** and **Matrix Factorization Approach** models are totally different, we can allocate different weights to these models to construct the final recommendation. 
      - Like 0.2 for **Popularity-based**, 0.2 for **Neighborhood-based**, 0.6 for overlap of **NMF** and **TruncatedSVD**.
- Next step
  - Besides the insights mentioned above, I think there are aspects we can further dive deep, like:
    - As we have huge amounts of users, we can try to perform clustering to all users, cluster users with high similarities into the same cluster, which allows us to perform different recommendation algorithm to different clusters, more efficient and more targeted.
    - Develop 'dislike' feature which allow users to flag songs they don't like, so that our recommender will not recommend the disliked songs again, on the other hand, our recommender can avoid recommending songs with high similarities with disliked songs.
    - Our recommendation system should also consider the style of the song, such as recommending rock or pop music in working hours, recommending light music or antiques in evening time, etc.




  

