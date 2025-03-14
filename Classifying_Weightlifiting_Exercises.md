---
title: "Classifying Weightlifting Exercises"
author: "Julian Simington"
date: "2025-01-20"
output:
  html_document:
    keep_md: true
---

## Introduction
\
The goal of this project is to classify human activity using experimental data. Data from the Human Activity Recognition (HAR) project was used. Subjects performed  weight lifting exercises described by the `classe` variable with levels `A`, `B`, `C`, `D`, and `E`. Level `A` corresponded to the exercise being performed correctly, while other levels corresponding to different methods of incorrect performance. All levels were performed with guidance by an instructor.

See the README on my Github for a link to the paper.
  
  
## Summary
The final model chosen was a "naive" random forest model built using **5-fold cross validation on 52 unprocessed predictors**. The model **predicted `classe` with 99.5% accuracy in the test set (0.5% out of sample error)**. Additionally, this model predicted a separate validation set with 100% accuracy. Somewhat surprisingly, the model outperformed a random forest preprocessed using PCA. See the *Feature Selection* section for additional discussion.
  
The final model was built by first cleaning the data set. The data included 100 variables which contained ~ 98% missing data. These variables were removed rather than attempting imputations. Additionally, 7 identifier variables not needed for classification were removed. The remaining 52 predictors were used to predict `classe`.

Features were explored using a correlation matrix, heatmap, and Principal Component Analysis (PCA). This analysis revealed highly correlated features within the data. Thenaive random forest model (99.5%) was built *after* evaluating a random forest using PCA preprocessing (98% accuracy) and a Gradient Boosted Machine (~80% accuracy). The high correlation among features suggested the use of PCA may be effective but this was not the case in practice.

Model stacking was considered, but was not explored due to limited computing power and the high accuracy of the naive random forest model.

## Package dependencies

The following packages are used-- `dplyr`, `ggplot2`, `caret`, `pROC`, `corrplot`, `parallel`, and `doparallel`. Installation code is suppressed.



## Loading Data
  
The data is downloaded if necessary. Code is suppressed.




## Data Cleaning
The code below creates training, test, and validation data sets. The validation data set was used for quiz predictions. The `train_data` set was 19622 x 160 while the `quiz_validation_data` set was 20 x 160. Ultimately, **53 features were selected** from the original data set for modeling, then`train_data` was split into train and test sets.  
\
Options for dimension reduction were explored since the data contained 160 features. However 100 of these features had significant amounts of missing data (~ 98% missing data). Of these, 67 columns contained *exactly* **19216** NAs and 37 columns contained *exactly* **19216** blank values. These columns were removed from the data rather than attempting imputations due to the large amount of missing data. Histograms showing the distribution of `NAs` and blanks are displayed below.  
\
The resulting data contained 60 variables. These variables included 7 identifiers, including time windows. Time information suggests modeling the data as a time series. However, the goal is to identify *which* weightlifting exercise is performed. While the data was collected in time windows, subjects were instructed to perform specific movements that do not have any relation to the time window in which they were performed. As such all 7 identifiers were also removed.  
  
The code below cleans the data as described above. The resulting data contained 53 columns-- 52 features and 1 response (`classe`). Additionally `train_data` is split into train and test sets. The seed was set to **343** for reproducibility.


``` r
train_data <- read.csv('Weightlifting_Data_train.csv', header = TRUE) # split into train/test data later
quiz_validation_data <- read.csv('Weightlifting_Data_test.csv', header = TRUE) # validation data set (quiz)


col_NAs <- sapply(train_data, function(x){sum(is.na(x))}) # get count of NAs in each column

# sum(col_NAs > 19000) # 67 predictors have more than 19000 NAs
# sum(col_NAs == 19216) # 67 predictors have exactly 19,216 NA values

train_data_clean <- train_data[,col_NAs<19216] # subset data with
quiz_validation_data_clean <- quiz_validation_data[,col_NAs<19216]


# sum(train_data_clean$kurtosis_roll_belt == "") # also 19216 blank entries


feature_class <- sapply(train_data_clean,class) # get classes of remaining features
# get counts of blanks for character
feature_char <- feature_class[feature_class == 'character'] # names of character features
col_blanks <- sapply(train_data_clean,function(x){sum(x=="")})

# sum(col_blanks > 19000)  # we see the 19216 number again
# sum(col_blanks == 19216) # 19216 columns have 19216 blanks

par(mfrow = c(1,2))
hist(col_NAs) # NAs histogram
hist(col_blanks) # blanks histogram
```

![](Classifying_Weightlifiting_Exercises_files/figure-html/data_clean-1.png)<!-- -->

``` r
train_data_feat <- train_data_clean[,col_blanks < 19216]  # remove high blank count columns
quiz_validation_data_feat <- quiz_validation_data_clean[,col_blanks < 19216] 


train_data_feat <- train_data_feat |> select(-(X:num_window)) # remove identifier columns
quiz_validation_data_feat <- quiz_validation_data_feat |> select(-(X:num_window))


# further split "training" data into train/test
set.seed(343)
train_indices <- createDataPartition(train_data_feat$classe, p = 0.7, list = FALSE) # partition data
feat_train <- train_data_feat[train_indices,] # train data. 13737 x 53
feat_test <- train_data_feat[-train_indices,] # test data 5885 x 53
```

## Feature Selection/Dimension Reduction
  
The following methods were used to explore dimension reduction.
  

* Correlation matrix
* Heatmap 
* PCA pre-processing
  
A visualized correlation matrix and heatmap are displayed below. Feature names are not displayed in the correlation plot for improved readability.
  

![](Classifying_Weightlifiting_Exercises_files/figure-html/feat_select-1.png)<!-- -->![](Classifying_Weightlifiting_Exercises_files/figure-html/feat_select-2.png)<!-- -->

The correlation matrix indicates strong correlations between a number of predictors with dark blue and dark red circles representing strong positive and strong negative correlations, respectively. Note the diagonal is omitted. The heatmap further reinforces the presence of correlated predictors. The darker orange section in the middle/right portion corresponds to the 21 variables:



``` r
colnames(feat_train_mat)[train_heat$colInd][23:43] # orange block in heatmap
```

```
##  [1] "yaw_arm"              "roll_arm"             "accel_belt_x"        
##  [4] "pitch_arm"            "pitch_forearm"        "pitch_belt"          
##  [7] "gyros_dumbbell_z"     "gyros_forearm_z"      "gyros_forearm_y"     
## [10] "gyros_arm_x"          "gyros_arm_y"          "gyros_belt_z"        
## [13] "gyros_belt_x"         "gyros_belt_y"         "gyros_dumbbell_y"    
## [16] "gyros_forearm_x"      "gyros_arm_z"          "gyros_dumbbell_x"    
## [19] "total_accel_belt"     "total_accel_dumbbell" "accel_belt_y"
```


PCA preprocessing indicated that 25 principal components were needed to account for 95% of the variation within the training data. However, a number of features were not normally distributed. For example, within the training data the feature `roll_belt` had a bimodal distribution with large gaps. A model was fit using preprocessed PCA predictors; however ultimately it underperformed a naive model out of sample. It is possible this was due to the lack of normality in the predictors and data transformations (e.g. Box Cox) were appropriate. These plots were excluded to limitations on the permitted number of figures.

Note: 18 principal components captured 90% of the variance, 36 components captured 99% of the variance, and 52 components captured 100% of the variance in the training set. Again due to concerns with non-normality of features these values may not be reliable.


## Model Building
  
The following models were considered  using the training set
  
* Random Forest with PCA preprocessing (95% variation)
* Gradient Boosting with PCA preprocessing (95% variation)
* Random Forest with no preprocessing (naive)
  

The random forest model used 5-fold cross validation, setting aside 20% of the training data for testing. It was was chosen due to high accuracy and because a nonparametric method was advisable given the non-normality of some features. Overfitting was a concern; however, PCA was intended to mitigate the issue. and Gradient Boosting was also selected as an ensemble method in hopes of high accuracy. However both models were outperformed by the naive random forest model .

The code below trains each model. The models were very time consuming for my computer to run so only the code for the naive random forest has been run. Code for the other models is commented out and the results are summarized below.


``` r
# set up parallel cluster processing to speed up random forest
cluster <- makeCluster(detectCores() - 1) # convention to leave 1 core for OS
registerDoParallel(cluster)

fitControl1 <-  trainControl(method = "cv",  number = 5, allowParallel = TRUE) # 5 fold cross validation


# pca_pp_train95 = preProcess(feat_train, method = "pca", thresh = 0.95)
# pca_train_feat <- predict(pca_pp_train95, feat_train) # apply pca on features

# classe_model_rf <- train(classe ~ . , method = "rf", data = pca_train_feat, # fit pca random forest
#                         trControl = fitControl1)


# pca_test_feat <- predict(pca_pp_train95, feat_test) # pca on test features



# classe_model_gbm <- train(classe ~ . , method = "gbm", data = pca_train_feat, # fit gbm
#                          trControl = fitControl1)



class_no_pp_model <- train(classe ~ . , method = "rf", data = feat_train,   # fit naive random forest
                                                       trControl = fitControl1)
```

```
## Warning in nominalTrainWorkflow(x = x, y = y, wts = weights, info = trainInfo,
## : There were missing values in resampled performance measures.
```

## Predictions/Error


``` r
# pca_test_feat <- predict(pca_pp_train95, feat_test) # apply pca to test features
#classe_rf_pred <- predict(classe_model_rf, pca_test_feat) # predict classe on test set
#98% accuracy on test sample


classe_no_pp_pred <- predict(class_no_pp_model ,feat_test)
conf_Mat <- confusionMatrix(classe_no_pp_pred, as.factor(feat_test$classe))
conf_Mat$overall[1] # 99.5% accuracy out of sample, 100% in sample
```

```
##  Accuracy 
## 0.9940527
```

  
##Conclusion  
  
The naive random forest model performed with 99.5% accuracy on the test set suggesting 0.5% average out of sample error. By comparison the 95% variation PCA model has 98.2% accuracy and the Gradient Boosted model had ~81% accuracy. Additionally the naive random forest model reported 100% accuracy on the quiz validation set.
