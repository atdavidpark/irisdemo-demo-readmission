# Using Realtime Machine Learning to reduce Readmission Risks

Patient Readmissions are said to be the "Hello World of Machine Learning" in Healthcare. On this demo, we use this problem to show how IRIS can be used to **safely build and operationalize** ML models for real time predictions and how this can be integrated into a random application. This **IRIS for Health** demo seeks to show how a full solution for this problem can be built. It shows:
* **IRIS Interoperability** extracting data from HL7v2 messages send by a simulated EMR
* **IRIS Database** storing the normalized data and helping with the data preparation steps. We use IRIS Analytics and a Cube for this purpose. The cube can be exposed as an [Analytical Base Table (ABT)](https://en.wikipedia.org/wiki/Analytical_base_table) for building the ML model.
* **Spark Cluster and a Zeppelin Notebook** being used to consume the data from the ABT in IRIS and build the model. The model is then exported as [PMML](https://en.wikipedia.org/wiki/Predictive_Model_Markup_Language).
* **IRIS Database running the model close to the data** - The PMML model is imported by IRIS and it is run close to the data. This allows us to:
  * Run the model with low latency since we don't have to call out to the Spark Cluster anymore
  * Run the model safely since we can guarantee that the same steps for data preparation for model building are being executed for operationalizing the model
  * Expose a simple REST service that allows someone to predict the risk of readmission without having access to all the data.
* **Synthea** - An example of loading FHIR bundles generated by Synthea into IRIS. We pre-load 5000 patients worth of data generated by [Synthea](https://synthetichealth.github.io/synthea/) in order to build the readmission risk model using the Spark Cluster. As you will see bellow, the first version of this demo was built based on 11 million real encounters from a region in the US. Only after we saw it was possible to build a model and that this model could perform better than [LACE](https://www.mdcalc.com/lace-index-readmission) we decided to publish this demo as open source using synthetic data. So, this demo is real. The ML model that we are building may not be the best (because synthetic data has its problems), but the tooling and methodology are real and they work on real data pretty well.
* **LACE** - As a baseline, we compare the risk predictions from our model to the prediction returned by LACE. The EMR simulation will allow you to see that and realize that the ML model can perform better.

The picture bellow shows the architecture of this demo:

![Demo Landing Page](https://raw.githubusercontent.com/intersystems-community/irisdemo-demo-readmission/master/DemoArchitecture.png?raw=true)

In order to predict if a patient is going to be readmitted in the next 30 days after being discharged from a hospitalization, we need data from the current hospitalization and from all previous encounters. That can be proven difficult because of the variety of healthcare data and standards (HL7v2, CDA documents, IHE profiles, FHIR, ASTM, etc.). All the data sent using all these different standards must be extracted from these messages and documents and stored into a normalized data lake. 

Once the data is stored, the machine learning model can be built. A data engineer or a data scientist will start with the data preparation steps. They will transform the normalized data model into a single table that we call ABT (Analytical Base Table). Machine Learning algorithms need this to make sense of the data and generate useful models for predicting readmissions. 

Leaving this data preparation step to be done by the data scientist/engineer on his tool of choice is a common option. But that implies:
* **Replicating Data** - replicating real clinical data to these third party tools 
* **Operation Costs** - when it is time to operationalize the model (use it in production) we will need to leave the machine learning platform (Spark cluster or other solution on some cloud vendor, Python scripts, R scripts, etc) up and running so we can run the model. That can be dangerous because of data privacy/security concerns and adds to cost to the final solution
* **Operation Latency** - Calling out to the ML platform to run the model adds more latency to the final solution. That doesn't impact the readmission problem but it may impact other problems such as preventing fraud on online transactions.

This demo uses docker compose which relies on a file called **docker-compose.yml** that describes the services we want to start. Our demo uses 8 services:

* **hisui** - this is the EMR (electronic medical record) simulation. It is not a real EMR. Its UI is built using Angular 8 and its backend in this case is an IRIS Database. We call the backend using REST services.
* **hisdb** - This is the backend of the EMR. It is a demo in of itself since it shows how to call IRIS using REST services from an Angular application and how changes on the SQL datamodel can trigger HL7 messages being queued up for sending to other systems.
* **risksrv** - This is the interoperability layer. It receives HL7v2 messages (and could potentially receive messages and documents in virtually all major healthcare standards). It extracts relevant information from the messages and store it on the normalized datalake that is on the **riskengine** container. It will also monitor the risk scores (both LACE and the ML model risk) and trigger a workflow with the care team if a patient is discharged with a high risk of readmission. Here we are taking a conservative approach. If LACE or ML says the patient is at risk, we assume the patient is at risk and we trigger the workflow. This allows for a smooth incorporation of ML into the process with minimal risk of letting patients at risk out.
* **riskengine** - This is the normalized data lake we use to aggregate data from all the data sources. We have pre-loaded it with 5000 synthetic patients to simulate all these data sources and it is being fed by the EMR's data in realtime. When we discharge a patient, the HL7v2 message sent by the EMR will be parsed by **risksrv** and the data will be stored here so that the readmission scores can be re-computed considering the new information. So, yes, it is on this box where we compute both LACE and where we run the ML model close to the data. Data for the ML model is aggregated into a cube and exposed to the Spark Cluster as an ABT (analytical base table). This box also monitors a folder for new PMML files generated by spark (with new models to be imported).
* **zeppelin** - This is the notebook we use to drive the Spark Cluster and build the ML models and export them as PMML.
* **sparkmaster** - This is the Master of the Spark Cluster we use to build the model.
* **sparkworkerN** - These are spark worker nodes. The idea is to simulate a real spark cluster so that architects can understand how this could be setup on the real world.

## How to run the demo

**WARNING: If you are running on a Mac or Windows, you must give Docker at least 5888MB of RAM for this demo to run properly. Also, please check this [troubleshooting](https://github.com/intersystems-community/irisdemo-base-troubleshooting) document in case you find problems starting the demo with docker-compose. Disk space available for the docker VM is the most common cause for trouble.**

To run the demo on your PC, make sure you have git and Docker installed on your machine. 

Clone this repository to your local machine to get the entire source code. Don't worry, you don't need to rebuild the demo from its source. It is just easier this way. After cloning the repo, change to its directory and start the demo:

```bash
git clone https://github.com/intersystems-community/irisdemo-demo-readmission
cd irisdemo-demo-readmission
docker-compose up
```

When starting, you will see lots of messages from all the containers that are starting. That is fine. Don't worry!

When it is done, it will just hang there, without returning control to you. That is fine too. Just leave this window open. If you CTRL+C on this window, docker compose will stop all the containers and stop the demo.

After all the containers have started, open a browser at [http://localhost:9092/csp/appint/demo.csp](http://localhost:9092/csp/appint/demo.csp) to see the landing page of the demo. When requested, use the credentials SuperUser/sys to log in. 

A video about this demo will be published here soon. But for now, you can press at the **Instructions** button at the bottom right of the page to see one example of how to use the demo. The demo has different stories for different publics. 

I like to open EMR's UI and click on the "Patient Census" tab to see the three patients that are hospitalized and their risk scores. I show that ML did a better job than lace, scoring a patient with a 55% risk of readmission while LACE was scoring around 7%. You can click on the LACE score to see its individual components and an explanation about how the score relates to a % risk of readmission. I like to click on the ML % score and show that I can simulate sending the patient to a Nursing Home to see the impact of this decision on the ML % risk score in real time. It reduces the risk to 30% or so. Then I actually discharge the patient to a Nursing Home. This is a very high level demo and you could stop there if you want to. Or you can open other parts of IRIS and show how we got the HL7 discharge message on the **risksrv** layer and extracted the data and used the ML model on **riskengine** to compute the risk score and decide if we should alert the care team and add the patient to a program to be monitored. There are plenty to show and many possible stories!

When you are done, go back to that terminal and enter CTRL+C. You may also want to enter with the following commands to stop containers that may still be running and remove them:

```bash
docker-compose stop
docker-compose rm
```

This is important, specially if you have other compositions on your machine.

## How to reset the demo?

You discharged the patient on the EMR and now it is gone! How can you repeat this to run the demo again?

It's easy! You can click on the "Log Out" link at the top-right of the EMR's UI. It will reset the demo for you, putting the patient back at the hospital. Or you can just close the EMR and open it back again. Your patient will be back there.

Another way of resetting the demo is to stop the docker composition, remove the containers and start it again.

## What if I want to rebuild the demo on my PC?

You can do it by running the ./build.sh script. The build will take several minutes to finish because loading the almost six thousand patient FHIR bundles generated by Synthea into IRIS takes time. These files are pre-generated and packed into an image of its own. It is loaded during the build of the image-riskengine that is a multi-build image.

## What is the final ML model that is being used and how many features does it take?

If you want to see how many features we are feeding into the model, run the following query on the APPINT namespace on the Data Warehouse box (the open data box):

```SQL
call PublishedABT.MLEncounterGetFeatures()
```

The final model being used is a **Random Forest**. The list bellow has the total number of features fed into the model (44) and the bold ones are the ones that the algorithm has actually chosen as relevant (35) for the readmission problem on this dataset: 
1. **AdmitReason**
2. **Com_ANY_MALIGNANCY_INCLUDING_LYMPHOMA_AND_LEUKEMIA_EXCEPT_MALIGNANT_NEOPLASM_OF_SKIN**
3. **Com_CEREBROVASCULAR_DISEASE**
4. **Com_CHRONIC_PULMONARY_DISEASE**
5. **Com_DEMENTIA**
6. **Com_DIABETES_WITHOUT_CHRONIC_COMPLICATION**
7. Com_HEMIPLEGIA_OR_PARAPLEGIA
8. **Com_MYOCARDIAL_INFARCTION**
9. **Com_PERIPHERY_VASCULAR_DISEASE**
10. Com_MILD_LIVER_DISEASE
11. **Com_RENAL_DISEASE**
12. **CurCom_ANY_MALIGNANCY_INCLUDING_LYMPHOMA_AND_LEUKEMIA_EXCEPT_MALIGNANT_NEOPLASM_OF_SKIN**
13. **CurCom_CEREBROVASCULAR_DISEASE**
14. CurCom_CHRONIC_PULMONARY_DISEASE
15. CurCom_DEMENTIA
16. CurCom_MILD_LIVER_DISEASE
17. CurCom_MYOCARDIAL_INFARCTION
18. CurCom_PERIPHERY_VASCULAR_DISEASE
19. **CurCom_RENAL_DISEASE**
20. **DxAgeGroup**
21. **DxDischargeLocation**
22. DxEncounterType
23. **DxGenderViaPatient**
24. **LOS**
25. **MxAgeDischarged**
26. **MxAlcohol**
27. **MxDrugs**
28. **MxEncounterEndYear**
29. **MxEncounterStartYear**
30. **MxEndDateDayOfMonth**
31. **MxEndDateDayOfWeek**
32. **MxEndDateMonth**
33. **MxExSmoker**
34. **MxNeverSmoked**
35. **MxNumAdmitsOneMonth**
36. **MxNumAdmitsSixMonth**
37. **MxNumAdmitsThreeMonth**
38. **MxNumAdmitsTwelveMonth**
39. **MxNumDiagOneMonth**
40. **MxNumDiagTwelveMonth**
41. MxSmoker
42. **MxStartDateDayOfMonth**
43. **MxStartDateDayOfWeek**
44. **MxStartDateMonth**

You will notice that not all commorbidities are appearing. That is because our synthetic database didn't have those commorbidities. It is always good to remember thought, that this model was initially built with real data and validated with it. Then we deleted the data and loaded synthetic data on the same tables and retrained the model with it. The results were good enough for us to ship the demo and keep telling the story. You could try loading your data on the data warehouse and rebuilding the model.

Curious things:
* It seems that it doesn't matter if you are not a smoker anymore (MxSmoker=1). If you have ever been a smoker (MxExSmoker=1 or MxNeverSmoked=0) that will affect your risk.
* It seems that the gender of the patient was relevant for the readmission problem 

# Other demo applications

There are other IRIS demo applications that touch different subjects such as NLP, ML, Integration with AWS services, Twitter services, performance benchmarks etc. Here are some of them:
* [HTAP Demo](https://github.com/intersystems-community/irisdemo-demo-htap) - Hybrid Transaction-Analytical Processing benchmark. See how fast IRIS can insert and query at the same time. You will notice it is up to 20x faster than AWS Aurora!
* [Fraud Prevention](https://github.com/intersystems-community/irisdemo-demo-fraudprevention) - Apply Machine Learning and Business Rules to prevent frauds in financial services transactions using IRIS.
* [Twitter Sentiment Analysis](https://github.com/intersystems-community/irisdemo-demo-twittersentiment) - Shows how IRIS can be used to consume Tweets in realtime and use its NLP (natural language processing) and business rules capabilities to evaluate the tweet's sentiment and the metadata to make decisions on when to contact someone to offer support.
* [HL7 Appointments and SMS (text messages) application](https://github.com/intersystems-community/irisdemo-demo-appointmentsms) -  Shows how IRIS for Health can be used to parse HL7 appointment messages to send SMS (text messages) appointment reminders to patients. It also shows real time dashboards based on appointments data stored in a normalized data lake.
* [The Readmission Demo](https://github.com/intersystems-community/irisdemo-demo-readmission) - This demo. Patient Readmissions are said to be the "Hello World of Machine Learning" in Healthcare. On this demo, we use this problem to show how IRIS can be used to **safely build and operationalize** ML models for real time predictions and how this can be integrated into a random application. This **IRIS for Health** demo seeks to show how a full solution for this problem can be built.

# Report any Issues
  
Please, report any issues on the [Issues section](https://github.com/intersystems-community/irisdemo-demo-readmission/issues).

# Change Log

Click [here](https://github.com/intersystems-community/irisdemo-demo-readmission/blob/master/CHANGELOG.md) to see the change log.
