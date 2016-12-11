Simple use-case to run ICU Mortality Prediction
-----------------------------------------------

A simplistic example of a spark-submit command to run the program to construct/store features, training the SVM using 70% of input data and generating AUC/ROC for testing set (30% of the input) in a local computer with Baseline (age, sex without SAPS generated from MIMIC codebase), Time-varying events (Lab, Diag, Drugs) and Notes (without Comorbidities generated from MIMIC codebase) using standard MIMIC files (keeping the MIMIC3 files name and format unchanged but can have small set of data)-

**Pre-requiste**: Apache Spark 2.0.0 is installed in the local machine.

Step-1: Compile the source code and create the fat .jar by running the following sbt commands -

    a. command => cd [sourcecode]     #changing the directory to the source-code root location
    b. command => sbt assembly        #Create the jar

    The mortality_prediction-assembly-1.0.jar will be created at [sourcecode]/target/scala-2.11/mortality_prediction-assembly-1.0.jar

Step-2: Run the following command -

       spark-submit --driver-memory 4G --executor-memory 4G \
       --master local \
       --class com.datalogs.main.Main [sourcecode]/target/scala-2.11/mortality_prediction-assembly-1.0.jar \
       --csv-dir "[MIMIC-input-file-location]" \
       --feature-dir "To-be-created-feature-location" \
       --output-dir "Output-location" \
       --stop-word-file "[sourcecode-location]/src/main/resources/stopWords.txt" \
       --no-saps-data \
       --no-comorbidities \
       --pipeline-stage 0

   The explanation of each command line parameters and output are available in the detailed section below.

Detailed ICU Mortality Prediction Setup Instructions
-------------------------------------------

1.  Download the MIMICIII Clinical Database from https://physionet.org/works/MIMICIIIClinicalDatabase/

2.  Load the MIMIC-III database to POSTGRES to generate the SAPS-score and Comorbidities using the following .sql scripts  of Mimic codebase at https://github.com/MIT-LCP/mimic-code/tree/master/buildmimic/postgres -

    2.1. Create Mimic Database using the following psql commands-

        CREATE USER mimic;
        ALTER USER mimic superuser;
        CREATE DATABASE mimic OWNER mimic;

    2.1. Run https://github.com/MIT-LCP/mimic-code/blob/master/buildmimic/postgres/postgres_create_tables.sql to create tables.

        E.g. psql -f postgres_create_tables.sql -U mimic

    2.2. Run https://github.com/MIT-LCP/mimic-code/blob/master/buildmimic/postgres/postgres_load_data.sql to load data.

        E.g. psql -f postgres_load_data.sql -U mimic -v mimic_data_dir='[MIMICIII-CSV-Files-location]'

    2.3. Run https://github.com/MIT-LCP/mimic-code/blob/master/buildmimic/postgres/postgres_add_indexes.sql to add indexes.

        E.g. psql -f postgres_add_indexes.sql -U mimic

    2.4. Run https://github.com/MIT-LCP/mimic-code/blob/master/buildmimic/postgres/postgres_add_constraints.sql to add constraints.

        E.g. psql -f postgres_add_constraints.sql -U mimic

3.  Generate the SAPSII score using MIMIC codebase -

    3.1. Create the following views with .sql (example below) at https://github.com/MIT-LCP/mimic-code/tree/master/etc/firstday -

        --  1) uofirstday - generated by urine-output-first-day.sql
        --  2) ventfirstday - generated by ventilated-first-day.sql
        --  3) vitalsfirstday - generated by vitals-first-day.sql
        --  4) gcsfirstday - generated by gcs-first-day.sql
        --  5) labsfirstday - generated by labs-first-day.sql

        E.g. psql -f urine-output-first-day.sql -U mimic

    3.2. Run the following .sql at https://github.com/MIT-LCP/mimic-code/blob/master/severityscores/sapsii.sql to create the SAPSII view.

        E.g. psql -f sapsii.sql -U mimic

4.  Derive the Comorbidities features using MIMIC codebase -

    4.1. Run the following .sql at https://github.com/MIT-LCP/mimic-code/blob/master/comorbidity/postgres/elixhauser-ahrq-v37-with-drg.sql to create the ELIXHAUSER_AHRQ materialized view. It creates 30 EH comorbidities.

        E.g. psql -f elixhauser-ahrq-v37-with-drg.sql -U mimic

5.  Generate the SAPSII and Comorbidities features in .CSV format from POSTGRES tables -

    5.1. Run the following query at psql prompt to dump or create the SAPSII.CSV file for all patients -

        COPY (SELECT * FROM sapsii) TO '[CSV FILE LOCATION]/SAPSII.csv' DELIMITER ',' CSV HEADER;

    5.2. Run the following query at psql prompt to dump or create the EHCOMORBIDITIES.csv file for all patients -

        COPY (SELECT * FROM ELIXHAUSER_AHRQ) TO '[CSV FILE LOCATION]/EHCOMORBIDITIES.csv'  DELIMITER ',' CSV HEADER;

6.  Remove special characters from the NOTEEVENTS.csv and LABEVENTS.csv files to avoid any potential error during feature creation -

        6.1. Run the script provided in the project codebase found at [code-root]/scripts/process_notes.sh to remove the special characters (e.g. ""\n" between two qutotes in a column) from NOTEEVENTS.csv. Run the following commands in this order -
            e.g.  chmod u+x [sourcecode-root]/scripts/process_notes.sh        #change the .sh file permission to  make it executable
                  mv NOTEEVENTS.csv NOTEEVENTS_ORG.csv                        #rename the NOTEEVENTS.csv file for further modification
                  [sourcecode-root]/scripts/process_notes.sh NOTEEVENTS_ORG.csv NOTEEVENTS.csv

        6.2. Run the script provided in the project codebase found at [code-root]/scripts/process_labresults.sh to remove the special characters (e.g. "," between two qutotes in a column) from LABEVENTS.csv. Run the following commands this in order -
            e.g.  chmod u+x [sourcecode-root]/scripts/process_labresults.sh   #change the .sh file permission to make it exexcutable
                  mv LABEVENTS.csv LABEVENTS_ORG.csv                          #rename the LABEVENTS.csv file for further modification
                  [sourcecode-root]/scripts/process_labresults.sh LABEVENTS_ORG.csv LABEVENTS.csv

7.  The ICU Mortality Prediction programs depends on the following file names (file-name should match as specified below) and and its .csv format (comma separated and first row having column names) with the following column names. The location of the .csv files can be passed as parameter to the main program (explained in the later part of the instruction steps)-

        7.1. File-name: PATIENTS.csv                     => Column-name: "ROW_ID","SUBJECT_ID","GENDER","DOB","DOD","DOD_HOSP","DOD_SSN","EXPIRE_FLAG"

        7.2. File-name: ICUSTAYS.csv                     => Column-name: "ROW_ID","SUBJECT_ID","HADM_ID","ICUSTAY_ID","DBSOURCE","FIRST_CAREUNIT","LAST_CAREUNIT","FIRST_WARDID","LAST_WARDID","INTIME","OUTTIME","LOS"

        7.3. File-name: CHARTEVENTS.csv                  => Column-name: "ROW_ID","SUBJECT_ID","HADM_ID","ICUSTAY_ID","ITEMID","CHARTTIME","STORETIME","CGID","VALUE","VALUENUM","VALUEUOM","WARNING","ERROR","RESULTSTATUS","STOPPED"

        7.4. File-name: LABEVENTS.csv                    => Column-name: "ROW_ID","SUBJECT_ID","HADM_ID","ITEMID","CHARTTIME","VALUE","VALUENUM","VALUEUOM","FLAG"

        7.5. File-name: DIAGNOSES_ICD.csv                => Column-name: "ROW_ID","SUBJECT_ID","HADM_ID","SEQ_NUM","ICD9_CODE"

        7.6. File-name: PRESCRIPTIONS.csv                => Column-name: "ROW_ID","SUBJECT_ID","HADM_ID","ICUSTAY_ID","STARTDATE","ENDDATE","DRUG_TYPE","DRUG","DRUG_NAME_POE","DRUG_NAME_GENERIC","FORMULARY_DRUG_CD","GSN","NDC","PROD_STRENGTH","DOSE_VAL_RX","DOSE_UNIT_RX","FORM_VAL_DISP","FORM_UNIT_DISP","ROUTE"

        7.7. File-name: NOTEEVENTS.csv                   => Column-name: "ROW_ID","SUBJECT_ID","HADM_ID","CHARTDATE","CHARTTIME","STORETIME","CATEGORY","DESCRIPTION","CGID","ISERROR","TEXT"

        7.9. File-name: SAPSII.csv                       => Column-name: subject_id,hadm_id,icustay_id,sapsii,sapsii_prob,age_score,hr_score,sysbp_score,temp_score,pao2fio2_score,uo_score,bun_score,wbc_score,potassium_score,sodium_score,bicarbonate_score,bilirubin_score,gcs_score,comorbidity_score,admissiontype_score

        7.10. File-name: EHCOMORBIDITIES.csv             => Column-name: subject_id,hadm_id,congestive_heart_failure,cardiac_arrhythmias,valvular_disease,pulmonary_circulation,peripheral_vascular,hypertension,paralysis,other_neurological,chronic_pulmonary,diabetes_uncomplicated,diabetes_complicated,hypothyroidism,renal_failure,liver_disease,peptic_ulcer,aids,lymphoma,metastatic_cancer,solid_tumor,rheumatoid_arthritis,coagulopathy,obesity,weight_loss,fluid_electrolyte,blood_loss_anemia,deficiency_anemias,alcohol_abuse,drug_abuse,psychoses,depression

        7.11. File-name: icu_mortality_test_patients.csv => Column-name: SUBJECT_ID

        *** Column names are not case-sensitive and they could be within quotes or without quotes.
        *** The column names are standard MIMICIII database column name except the SAPS and EHCOMORBIDITIES which are derived from MIMIC codebase.
        *** The icu_mortality_test_patients.csv is only required if a preselected list pf patients (subject_id only) are used as input test set. E.g. the patient ids given for Kaggle competition). This is optional.
        *** The SAPSII.csv, EHCOMORBIDITIES.csv, LABEVENTS.csv, DIAGNOSES_ICD.csv, and PRESCRIPTIONS.csv are optional for the main program and its inclusion or exclusion are controlled using command line parameter (explained in the later part of the instruction steps).

8.  Compile the source code and create the fat .jar by running the following sbt commands -

        8.1. command => cd [sourcecode]     #changing the directory to the source-code root location
        8.2. command => sbt assembly        #Create the jar

        The mortality_prediction-assembly-1.0.jar will be created at [sourcecode]/target/scala-2.11/mortality_prediction-assembly-1.0.jar

9.  Setting up the computation environment for running the program -

        9.1. Install Apache Spark 2.0.0. (Find download and install instruction at http://spark.apache.org/releases/spark-release-2-0-0.html)
        9.2. Either install Spark 2.0.0 in a Hadoop cluster (HDFS with Yarn in multi or single node) or in local machine.
        9.3. For full MIMIC-III database, it is recommended to use at least 10 model cluster with each node having at least 8GB memory in order to finish one complete run within 2 Hr.

10. Run the program using spark-submit command as follows -

        spark-submit --driver-memory 8G --executor-memory 8G \
        --master yarn \
        --num-executors 84 \
        --class com.datalogs.main.Main [sourcecode]/target/scala-2.11/mortality_prediction-assembly-1.0.jar \
        --csv-dir "[input-mimic-csv-files-location]" \
        --feature-dir "[program-output-feature-files-location]" \
        --test-feature-dir "[program-output-feature-files-location-for-test-datasets] \
        --output-dir "[program-output-location-to-store-ROC-values]" \
        --stop-word-file "[program-input-stop-word-file-path]" \
        --pipeline-stage 0

    OR

        spark-submit --driver-memory 8G --executor-memory 8G \
        --master local \
        --num-executors 84 \
        --class com.datalogs.main.Main [sourcecode]/target/scala-2.11/mortality_prediction-assembly-1.0.jar \
        --csv-dir "[input-mimic-csv-files-location]" \
        --feature-dir "[program-output-feature-files-location]" \
        --test-feature-dir "[program-output-feature-files-location-for-test-datasets] \
        --output-dir "[program-output-location-to-store-ROC-values]" \
        --stop-word-file "[program-input-stop-word-file-path]" \
        --no-saps-data \
        --no-comorbidities \
        --no-note-text \
        --no-event-data \
        --pipeline-stage 0

    The above program can be run -

            1. Either to construct all features (stored at "feature-dir") and run the SVMWithSGD using 70%-30% split of the entire input dataset for training-testing purpose with the ROC values of test patients stored at "output-dir". This will also print the AUC values for different models specified below at console.
            2. Or to construct the training features (stored at "feature-dir") from the entire input dataset (excluding the test set patients specified in the "icu_mortality_test_patients.csv" file at "csv-dir") and testing features (stored at "test-feature-dir") for the input test patients, then running a LogisticRegressionWithLBFGS and storing the mortality prediction probability result at "output-dir" directory. This is required also for the Kaggle competition part of this project.

    The outputs of this program are normally stored in "PART" files inside respective folder. You can merge all the part-files and store the entire output in a single file using the following command -

    For SVM (Option-1 described above) -

        e.g. cat [sourcecode]/output/svlight_inICU12Hr_baseline/part* > [sourcecode]/output/svlight_inICU12Hr_baseline.csv

    For LR (Option-2 described above) -

        e.g. echo 'SUBJECT_ID,InHospital_Expiry' &gt; [sourcecode]/output/svlight_inICU12Hr_baseline.csv  #This creates the cvs header required for Kaggle submission
             cat [sourcecode]/output/svlight_inICU12Hr_baseline/part* >> [sourcecode]/output/svlight_inICU12Hr_baseline.csv

    The program runs for the following models in 3 different outcomes -

        -- InICU12Hr, InICU24Hr, InICU48Hr, InICURetro     => For In-hospital or ICU mortality
        -- In30d12Hr, In30d24Hr, In30d48Hr, In30dRetro     => For 30-day Post Discharge mortality
        -- In1Yr12Hr, In1Yr4Hr, In1Yr48Hr, In1YrRetro      => For 1-Year Post Discharge mortality

    For each of the above model, the program runs for the following feature combinations -

        -- Baseline (including SAPS)
        -- Baseline (including SAPS) + Comorbidities
        -- Baseline (including SAPS) + Comorbidities + Dynamic-Events (Lab/Drug/Diag)
        -- Baseline (including SAPS) + Comorbidities + Notes
        -- All-Features (Baseline (including SAPS) + Comorbidities + Dynamic-Events (Lab/Drug/Diag) + Notes)

    In the output folder, “dynamic” means time-varying events from Lab, Drug and Diagnosis data.


   The command line parameters to run these program are -

    1.   "**--driver-memory**" and "**--executor-memory**" are optional and find details at http://spark.apache.org/docs/latest/configuration.html.

    2.   "**--master**" is required and can be set at "**local**" for local installation of spark or "**yarn**" (e.g. --master yarn)

    3.   "**--num-executors**" is optional and depends on cluster size and number of computational cores available. Find details at http://spark.apache.org/docs/latest/configuration.html.

    4.   "**--class**" is required to specify the fully-qualified-name of the "main" method in JAR and the location of the mortality_prediction-assembly-1.0.jar file.

    5.   "**--csv-dir**" is required to specify the location of MIMIC and other input files as required by this program.
            e.g. --csv-dir "[sourcecode]/MIMIC/input" (please don't put "/" at the end)

    6.   "**--feature-dir**" is required to specify the location of the program generated features file(s).
            e.g. --feature-dir "[sourcecode]/features" (please don't put "/" at the end)

    7.   "**--output-dir**" is required to specify the location of the output files generated by this program.
            e.g. --output-dir "[sourcecode]/features" (please don't put "/" at the end)

    7.   "**--test-feature-dir**" is optional and required to specify the location of the program generated test-features file (when a separate set of test patients are provided as input, e.g. Kaggle competion)created by this program.
            e.g. --test-feature-dir "[sourcecode]/test-features" (please don't put "/" at the end)

    8.   "**--stop-word-file**" is optional and required to specify the location of the stop-word file for the Latent Topic feature extraction. A default stop-word file (used during this project) is available at [sourcecode]/src/main/resources/stopWords.txt. But you can use a different one.
             e.g. --stop-word-file "[sourcecode]/src/main/resources/stopWords.txt"

    9.   "**--no-saps-data**" is optional and if specified, then SAPS data will be not used (or excluded) in constructing feature-sets.

    10.  "**--no-comorbidities**" is optional and if specified, then Comorbidities data will be not used (or excluded) in constructing feature-sets.

    11.  "**--no-note-text**" is optional and if specified, then Clinical Notes data will be not used (or excluded) in constructing feature-sets.

    12.  "**--no-event-data**" is optional and if specified, then time-varying event (Lab, Diag & Drug) data will be not used (or excluded) in constructing feature-sets.

    13.  "**--pipeline-stage**" is required to specify the workflow steps that the program will execute. There are 4 different options -

             --pipeline-stage 0 => This is to run the whole process in one run. This can be use for constructing features and running SVMWithSGD or LogisticRegressionWithLBFGS classification algorithm for different models depending on other parameters. If "--test-feature-dir" is specified then the program expects "icu_mortality_test_patients.csv" file at "csv-dir" and will run LogisticRegressionWithLBFGS, otherwise it will run SVMWithSGD (70%-30% splitting).
             --pipeline-stage 1 => For constructing features only for different models depending on other parameters. If "--test-feature-dir" is specified then the program expects "icu_mortality_test_patients.csv" file at "--csv-dir" and features will be created separately for training (at "feature-dir") and testing (at "test-feature-dir") set, sotherwise the features for entire input dataset set will be created at "feature-dir".
             --pipeline-stage 2 => To run the SVMWithSGD using the features constructed and stored at "feature-dir". This option assumes that the program was run before using "--pipeline-stage 1"  to create the features.
             --pipeline-stage 3 => To run the LogisticRegressionWithLBFGS using the training-set features constructed and stored at "feature-dir" and test patients feature constructed and stored at "test-feature-dir". This option assumes that the program was run before using "--pipeline-stage 1" to create the features for training-set and test-set inputs.

**Presentation Link**: https://www.youtube.com/watch?v=xRF_mo9GjBs