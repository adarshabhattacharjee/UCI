# UCI

To work with the dataset download it and then add the file in the codes provided.(https://archive.ics.uci.edu/dataset/296/diabetes+130-us+hospitals+for+years+1999-2008)


###PREPROCESSING
1. Working with the "?"

If it stays panda treats it as a real value which makes the missing record to be zero as it has some value. Therefore, it needs to be removed.

2. Removal of expired/hospice patients and invalid gender rows

- dead/hospice people are removed cause they cannot be readmitted.
Therefore prediction wont be used on them
- Invalid gender (3 rows): The model cannot learn anything reliable from such little information. thereby removing them will make the dataset cleaner.

3. Dropping weight, examide, citoglipton, encounter id
- Weight is recorded only fro 3 percent of the population
- No patient was ever given the drug examide or citoglipton
- Every patient has a unique id which is not going to help predict anything

4. Defining the target

The target is readmission before 30 days, We give the same value to the readmitted and the people who do not come back as we cannot predict beyond 30 days.

5. Removal of ambiguous parts
- "NotTested" for A1Cresult and max_glu_serum (It does not tell how the patient was managed.)
- "Unknown" for race, payer_code, medical_specialty
- Keeping only the top 10 medical speciality as there are 72 and top 10 cover most of them, and grouping the rest as others.(includes unknown which is around 49 percent)
It prevents some problems like overfitting.
- Merging the "NULL / Not Available / Not Mapped" ID codes
- Converting ID columns to strings since they are labels and not quantities

6. Group ICD-9(correspond to body systems) diagnosis codes into about 9 categories
- Mapped 716-789 unique codes per column into Circulatory, Respiratory, Digestive, Diabetes, Genitourinary, Injury, Musculoskeletal, Neoplasms, Other, and Unknown.
- Grouping uses clinical knowledge , so related conditions share statistical strength. This is the same grouping used in the original study by Strack et al. (2014), so your results are comparable to the literature.
7. Converting the age brackets into the midpoint of the age bracket (in the interval of 10)
8. It groups the patients who came more than one time and then assign the patients to train(80.1 percent) ot test(19.9 percent)

    WHY?
    - If the same person goes several times the model learns the pattern of that person. If an unknown person is to be tested upon it would thereby lead to a worse accuracy.
    - Grouping and later spliting proves that every patient is a stranger to the model.

2. Target variable 
The raw label readmitted has three classes. I define a binary target readmit_30 = 1 if readmitted == "<30", otherwise 0, so that "NO" and ">30" are both negatives.

We were given the target variable as readmission within 30 days. If the patient comes back after 30 days it is equivalent to the patient not coming at all because the model focuses only on the patients who are coming again before 30 days. The model is not required to predict how many people come after 30 days.

4. Algorithm choice and justification 

I chose gradient-boosted decision trees as the main model, with regularised logistic regression as an interpretable baseline. 

Why Boosted Trees Are the Best Fit for This Data:
- The dataset contains a mix of categories, ratings, and weird numbers. Decision trees work by cutting data at specific cutoff points. 
- Boosted trees spot combined patterns automatically
- There are hundreds of individual options like for the specialities and the medicines. Trees handle this safely by only paying attention to a specific code if there are enough patients in that group to make a reliable decision.
- There is a risk the model might memorize random noise instead of real patterns. To keep it stable, the trees are kept shallow, requiring at least 50 patients per group, making changes gradually, and stopping training early if performance stops improving. 

Only a small percentage of patients get readmitted. By giving class weightage to those readmitted patients during training, the model learns to actively look for high-risk warning signs rather than guessing no readmission every time.

3. Clinical considerations
In a clinical setting the model must flag the patients if the risk is too much and divide them on the basis of how susceptible they are to diabetes. Because class weighting inflated the probabilities, I had to add a separate isotonic-regression calibration step.


4. Evaluation methodology 
Metrics suited to healthcare risk prediction
- AUROC: It measures how well the model ranks patients who are readmitted versus those who are not.
- AUPRC: Since readmissions are relatively rare, this metric looks specifically at how well the model catches positive cases. 
A completely random guess would score around 11.4% (the average readmission rate). 
- Our score of 0.24 means the model is about twice as good as random guessing at spotting high-risk patients. 
- Calibration: We use a reliability curve and Brier score to verify that the model's numbers mean what they say. 
If the model flags a patient with 28% chance of returning, that estimate needs to be real. 
Threshold-based measures at a stated operating point:
- set thresholds based on the hospital's actual capacity. 
- If a care team only has the staff to follow up with the top 10% highest-risk patients, we tune the model to pick out that top 10%. I even added 5,10,20 and 30.


How We Split the Data to Avoid Cheating
- Splitting by Patient, Not by Visit: 
Since many patients visited the hospital multiple times (71,518 unique patients accounted for 101,766 total visits), splitting the data randomly by individual visits would cause overfitting as the model will learn from the individual patterns and predict based off of that and thereby decreasing the accuracy when we give strangers as input.

To prevent this, I split the data by patient. I set aside 80% of unique patients for training and locked away 20% for final testing. 
When testing and tuning the model internally, I used a fold that keeps every visit from a single patient together in one group, while making sure each group maintains a fair balance of readmitted and non-readmitted cases.

- Fair Uncertainty Calculations
When calculating error margins and confidence ranges, I used bootstrap.

This means when resampling the data to measure uncertainty, I picked whole patients (including all of their visits) rather than shuffling individual hospital visits. This ensures the error estimates accurately reflect real-world patient patterns.


Check whether a hospital's actual readmission rates match what the model predicts.
 A model can usually rank high-risk patients correctly at a new hospital, but its calculated percentages might be off because base readmission rates differ from site to site.
