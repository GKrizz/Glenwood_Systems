Day 344_QRDA Report Documentation


Hi Paddy,

       I have generated QRDA-1 for NOVtember 2026 and uploaded in below mentioned SFTP.  Please check. Kindly let us know if any concerns.


Date Range :: 10/01/2026 - 10/31/2026

SFTP credentials:-

sftp -oPort=22 paddy.g@ftp.spectramd.com

SFTP paths:-

FCM :- /First Care Medical

MID :- /Midwest Infectious Disease

SHIW :- /Stayhome Iwill

GIP:-  /GI Physicians
 
Thanks&Regards,
Gobala Krishnan



---------------------------------------------------------------Steps-----------------------------------------------------

1. BACKUP QUERY

Apply the below fix in mentioned Account :::

FCM
GIP
MID
SHIW

Send the backup file in below SFTP path

sftp -oPort=8444  glenwood@ftp.glaceemr.com:Temp/Kalai/MIPS_QRDA

MIPS:: Q1 reports

--------------------------------------------------------:::: BACKUP QUERY :::::------------------------------------------
check copy and revert query 


:::FCM :::: 11.63 ::::

\copy (select * from quality_measures_patient_entries where quality_measures_patient_entries_reporting_year=2026) to quality_measures_patient_entries_fcm_Jan_Mar_aco_bkp2026.csv with csv header;

delete from quality_measures_patient_entries where quality_measures_patient_entries_reporting_year=2026;

\copy (select * from macra_measures_rate where macra_measures_rate_reporting_year=2026) to macra_measures_rate_fcm_Jan_Mar_aco_bkp2026.csv with csv header;

delete from macra_measures_rate where macra_measures_rate_reporting_year=2026;


-------------------------------------------------------------------------------------------------------------------------
:::GIP :::: 11.42 ::::

\copy (select * from quality_measures_patient_entries where quality_measures_patient_entries_reporting_year=2026) to quality_measures_patient_entries_gip_Jan_Mar_aco_bkp2026.csv with csv header;

delete from quality_measures_patient_entries where quality_measures_patient_entries_reporting_year=2026;

\copy (select * from macra_measures_rate where macra_measures_rate_reporting_year=2026) to macra_measures_rate_gip_Jan_Mar_aco_bkp2026.csv with csv header;

delete from macra_measures_rate where macra_measures_rate_reporting_year=2026;


-------------------------------------------------------------------------------------------------------------------------
:::MID :::: 11.64 ::::

\copy (select * from quality_measures_patient_entries where quality_measures_patient_entries_reporting_year=2026) to quality_measures_patient_entries_mid_Jan_Mar_aco_bkp2026.csv with csv header;

delete from quality_measures_patient_entries where quality_measures_patient_entries_reporting_year=2026;

\copy (select * from macra_measures_rate where macra_measures_rate_reporting_year=2026) to macra_measures_rate_mid_Jan_Mar_aco_bkp2026.csv with csv header;

delete from macra_measures_rate where macra_measures_rate_reporting_year=2026;


-------------------------------------------------------------------------------------------------------------------------
:::SHIW :::: 11.92 ::::

\copy (select * from quality_measures_patient_entries where quality_measures_patient_entries_reporting_year=2026) to quality_measures_patient_entries_shiw_Jan_Mar_aco_bkp2026.csv with csv header;

delete from quality_measures_patient_entries where quality_measures_patient_entries_reporting_year=2026;

\copy (select * from macra_measures_rate where macra_measures_rate_reporting_year=2026) to macra_measures_rate_shiw_Jan_Mar_aco_bkp2026.csv with csv header;

delete from macra_measures_rate where macra_measures_rate_reporting_year=2026;


After checking copy , delete and revert query create fix in crm 

<img width="1293" height="628" alt="image" src="https://github.com/user-attachments/assets/7c78d739-c1eb-4166-9560-7f6d62646c50" />
<img width="1304" height="493" alt="image" src="https://github.com/user-attachments/assets/52c911a8-f1e0-4feb-8e3a-ab70f2ba18a0" />




2. CONFIGURE----------------------------------------------

Controlling High Blood Pressure
Diabetes: Glycemic Status Assessment Greater Than 9%
Preventive Care and Screening: Screening for Depression and Follow-Up Plan
Breast Cancer Screening


3. RUN BATCH----------------------------------------------


mode=3,reportingYear=2026,quartzId=vnzopk_fcm,accid=fcm,isMonthlyReport=false

mode=3,reportingYear=2026,quartzId=smqtil_gip,accid=gip,isMonthlyReport=false

mode=3,reportingYear=2026,quartzId=tkrwz_mid,accid=mid,isMonthlyReport=false

mode=3,reportingYear=2026,quartzId=plxome_shiw,accid=shiw,isMonthlyReport=false


----------------------------------------------------------

Hi Team,

Please provide the labs3 Console connection to my system (IP 183). Kindly let us know if any concern.

         Reason: For QRDA1 report.

Thanks,
Gobala Krishnan


4. LABS3 --> QRDA1----------------------------------------FCM---------------------------------------------------------- 

cd /mnt1/vs13shared/fcm/log/mu/QRDAFiles/QRDA_I/2026/'Oluremi Ojo M.D.' - Complete

Provider : Oluremi Ojo M.D.


236 CMS165v13 	- Controlling High Blood Pressure ---> - 107
001 CMS122v13 	- Diabetes: Glycemic Status Assessment Greater Than 9% ---> FCM- 45
134 CMS2v14 	- Preventive Care and Screening: Screening for Depression and Follow-Up Plan ---> FCM- 184
112 CMS125v13 	- Breast Cancer Screening ----> FCM- 53

sftp -oPort=8444  glenwood@ftp.glaceemr.com:Temp/Kalai/MIPS_QRDA/FCM
Gl3nW00d@$3curE$FTP!

put CMS125v13.zip
put CMS122v13.zip
put CMS165v13.zip
put CMS2v14.zip

rename CMS125v13.zip FCM_NOV_Breast_Cancer_CMS125v13.zip
rename CMS122v13.zip FCM_NOV_HbA1c_CMS122v13.zip
rename CMS165v13.zip FCM_NOV_High_Blood_Pressure_CMS165v13.zip
rename CMS2v14.zip FCM_NOV_Screening_for_Depression_CMS2v14.zip


----------------------------------------------------------MID----------------------------------------------------------

cd /mnt/vs22shared/mid/log/mu/QRDAFiles/QRDA_I/2026/'Solomon Beraki MD FACP CWS' - Complete

Provider : Solomon Beraki MD FACP CWS

236 CMS165v13 	- Controlling High Blood Pressure ---> MID- 2
001 CMS122v13 	- Diabetes: Glycemic Status Assessment Greater Than 9% ---> MID- 0
134 CMS2v14 	- Preventive Care and Screening: Screening for Depression and Follow-Up Plan ---> MID- 45
112 CMS125v13 	- Breast Cancer Screening ----> MID- 10

sftp -oPort=8444  glenwood@ftp.glaceemr.com:Temp/Kalai/MIPS_QRDA/MID
Gl3nW00d@$3curE$FTP!

put CMS125v13.zip
put CMS122v13.zip
put CMS165v13.zip
put CMS2v14.zip

rename CMS125v13.zip MID_NOV_Breast_Cancer_CMS125v13.zip
rename CMS122v13.zip MID_NOV_HbA1c_CMS122v13.zip
rename CMS165v13.zip MID_NOV_High_Blood_Pressure_CMS165v13.zip
rename CMS2v14.zip MID_NOV_Screening_for_Depression_CMS2v14.zip


----------------------------------------------------------SHIW----------------------------------------------------------

cd /mnt/vs22bshared/shiw/log/mu/QRDAFiles/QRDA_I/2026/'Radharamanamurthy Gokula M.D' - Complete

Provider : Radharamanamurthy Gokula M.D

236 CMS165v13 	- Controlling High Blood Pressure ---> Console-36
001 CMS122v13 	- Diabetes: Glycemic Status Assessment Greater Than 9% ---> SHIW-13
134 CMS2v14 	- Preventive Care and Screening: Screening for Depression and Follow-Up Plan ---> Console-148 
112 CMS125v13 	- Breast Cancer Screening ----> -36





sftp -oPort=8444  glenwood@ftp.glaceemr.com:Temp/Kalai/MIPS_QRDA/SHIW
Gl3nW00d@$3curE$FTP!

put CMS125v13.zip
put CMS122v13.zip
put CMS165v13.zip
put CMS2v14.zip

rename CMS125v13.zip SHIW_NOV_Breast_Cancer_CMS125v13.zip
rename CMS122v13.zip SHIW_NOV_HbA1c_CMS122v13.zip
rename CMS165v13.zip SHIW_NOV_High_Blood_Pressure_CMS165v13.zip
rename CMS2v14.zip SHIW_NOV_Screening_for_Depression_CMS2v14.zip


----------------------------------------------------------GIP----------------------------------------------------------

cd /mnt/vs23shared/gip/log/mu/QRDAFiles/QRDA_I/2026/'Ven Kottapalli MD, CNSP' - Complete

Provider : Ven Kottapalli MD, CNSP

236 CMS165v13 	- Controlling High Blood Pressure ---> GIP-6
001 CMS122v13 	- Diabetes: Glycemic Status Assessment Greater Than 9% ---> GIP-5
134 CMS2v14 	- Preventive Care and Screening: Screening for Depression and Follow-Up Plan ---> Console-188
112 CMS125v13 	- Breast Cancer Screening ----> GIP-35



001: Diabetes: Hemoglobin A1C (HbA1c) Poor Control (CMS122v13)
236: Controlling High Blood Pressure (CMS165v13)
134: Preventive Care and Screening: Screening for Depression and Follow-up Plan (CMS2v14)
112: Breast Cancer Screening (CMS125v13)

        
sftp -oPort=8444  glenwood@ftp.glaceemr.com:Temp/Kalai/MIPS_QRDA/GIP
Gl3nW00d@$3curE$FTP!

put CMS125v13.zip
put CMS122v13.zip
put CMS165v13.zip
put CMS2v14.zip

rename CMS125v13.zip  GIP_NOV_Breast_Cancer_CMS125v13.zip
rename CMS122v13.zip  GIP_NOV_HbA1c_CMS122v13.zip
rename CMS165v13.zip  GIP_NOV_High_Blood_Pressure_CMS165v13.zip
rename CMS2v14.zip  GIP_NOV_Screening_for_Depression_CMS2v14.zip





5. ----------------------------------------:::: REVERT QUERY :::::------------------------------------------------------

:::FCM :::: 11.63 ::::

delete from quality_measures_patient_entries where quality_measures_patient_entries_reporting_year=2026;

delete from macra_measures_rate where macra_measures_rate_reporting_year=2026;

\copy quality_measures_patient_entries from quality_measures_patient_entries_fcm_Jan_Mar_aco_bkp2026.csv with csv header;

select count (*) from quality_measures_patient_entries where quality_measures_patient_entries_reporting_year=2026;


-------------------------------------------------------------------------------------------------------------------------
:::GIP :::: 11.42 ::::

delete from quality_measures_patient_entries where quality_measures_patient_entries_reporting_year=2026;

delete from macra_measures_rate where macra_measures_rate_reporting_year=2026;

\copy quality_measures_patient_entries from quality_measures_patient_entries_gip_Jan_Mar_aco_bkp2026.csv with csv header;

select count (*) from quality_measures_patient_entries where quality_measures_patient_entries_reporting_year=2026;


-------------------------------------------------------------------------------------------------------------------------
:::MID :::: 11.64 ::::

delete from quality_measures_patient_entries where quality_measures_patient_entries_reporting_year=2026;

delete from macra_measures_rate where macra_measures_rate_reporting_year=2026;

\copy quality_measures_patient_entries from quality_measures_patient_entries_mid_Jan_Mar_aco_bkp2026.csv with csv header;

select count (*) from quality_measures_patient_entries where quality_measures_patient_entries_reporting_year=2026;


-------------------------------------------------------------------------------------------------------------------------
:::SHIW :::: 11.92 ::::

delete from quality_measures_patient_entries where quality_measures_patient_entries_reporting_year=2026;

delete from macra_measures_rate where macra_measures_rate_reporting_year=2026;

\copy quality_measures_patient_entries from quality_measures_patient_entries_shiw_Jan_Mar_aco_bkp2026.csv with csv header;

select count (*) from quality_measures_patient_entries where quality_measures_patient_entries_reporting_year=2026;


6. CONFIGURE------------------------FCM :::Oluremi Ojo M.D.

#1	CMS122v14	Diabetes: Glycemic Status Assessment Greater Than 9%
#130	CMS68v15	Documentation of Current Medications in the Medical Record
#317	CMS22v14	Preventive Care and Screening: Screening for High Blood Pressure and Follow-Up Documented
#134	CMS2v15		Preventive Care and Screening: Screening for Depression and Follow-Up Plan
#318	CMS139v14	Falls: Screening for Future Fall Risk
#236	CMS165v14	Controlling High Blood Pressure
#226	CMS138v14	Preventive Care and Screening: Tobacco Use: Screening and Cessation Intervention
#112	CMS125v14	Breast Cancer Screening

------------------------------------GIP :::Ven Kottapalli MD, CNSP

#238	CMS156v14	Use of High-Risk Medications in the Elderly
#1	CMS122v14	Diabetes: Glycemic Status Assessment Greater Than 9%
#130	CMS68v15	Documentation of Current Medications in the Medical Record
#317	CMS22v14	Preventive Care and Screening: Screening for High Blood Pressure and Follow-Up Documented
#134	CMS2v15		Preventive Care and Screening: Screening for Depression and Follow-Up Plan
#318	CMS139v14	Falls: Screening for Future Fall Risk
#236	CMS165v14	Controlling High Blood Pressure
#226	CMS138v14	Preventive Care and Screening: Tobacco Use: Screening and Cessation Intervention
#112	CMS125v14	Breast Cancer Screening

------------------------------------MID :::Solomon Beraki MD FACP CWS

#1	CMS122v14	Diabetes: Glycemic Status Assessment Greater Than 9%
#130	CMS68v15	Documentation of Current Medications in the Medical Record
#317	CMS22v14	Preventive Care and Screening: Screening for High Blood Pressure and Follow-Up Documented
#134	CMS2v15		Preventive Care and Screening: Screening for Depression and Follow-Up Plan
#318	CMS139v14	Falls: Screening for Future Fall Risk
#236	CMS165v14	Controlling High Blood Pressure
#226	CMS138v14	Preventive Care and Screening: Tobacco Use: Screening and Cessation Intervention
#112	CMS125v14	Breast Cancer Screening

------------------------------------SHIW :::Radharamanamurthy Gokula M.D

#1	CMS122v14	Diabetes: Glycemic Status Assessment Greater Than 9%
#130	CMS68v15	Documentation of Current Medications in the Medical Record
#317	CMS22v14	Preventive Care and Screening: Screening for High Blood Pressure and Follow-Up Documented
#134	CMS2v15		Preventive Care and Screening: Screening for Depression and Follow-Up Plan
#318	CMS139v14	Falls: Screening for Future Fall Risk
#236	CMS165v14	Controlling High Blood Pressure
#226	CMS138v14	Preventive Care and Screening: Tobacco Use: Screening and Cessation Intervention
#112	CMS125v14	Breast Cancer Screening

7. CALL 3 API----------::: 3 API :::-----------------------

http://emrsprings-dev.glaceemr.com/glaceemr_backend_stable/api/emr/glacemonitor/mipsperformance/calculateMIPSPerformance?reportingYear=2026&accountID=fcm&isMonthlyReport=false&dbname=fcm

http://emrsprings-dev.glaceemr.com/glaceemr_backend_stable/api/emr/glacemonitor/mipsperformance/calculateMIPSPerformance?reportingYear=2026&accountID=mid&isMonthlyReport=false&dbname=mid

http://emrsprings-dev.glaceemr.com/glaceemr_backend_stable/api/emr/glacemonitor/mipsperformance/calculateMIPSPerformance?reportingYear=2026&accountID=gip&isMonthlyReport=false&dbname=gip

http://emrsprings-dev.glaceemr.com/glaceemr_backend_stable/api/emr/glacemonitor/mipsperformance/calculateMIPSPerformance?reportingYear=2026&accountID=shiw&isMonthlyReport=false&dbname=shiw

8. SFTP---------------------------------------------------

cd /home/software/Documents/MIPS_Document/ACO/FCM
cd /home/software/Documents/MIPS_Document/ACO/MID
cd /home/software/Documents/MIPS_Document/ACO/SHIW
cd /home/software/Documents/MIPS_Document/ACO/GIP


sftp -oPort=8444  glenwood@ftp.glaceemr.com:Temp/Kalai/MIPS_QRDA
Gl3nW00d@$3curE$FTP!
ls -lrth

cd FCM

get FCM_NOV_Breast_Cancer_CMS125v13.zip
get FCM_NOV_HbA1c_CMS122v13.zip
get FCM_NOV_High_Blood_Pressure_CMS165v13.zip
get FCM_NOV_Screening_for_Depression_CMS2v14.zip


cd ..
cd MID

get MID_NOV_Breast_Cancer_CMS125v13.zip
get MID_NOV_High_Blood_Pressure_CMS165v13.zip
get MID_NOV_Screening_for_Depression_CMS2v14.zip


cd ..
cd SHIW

get SHIW_NOV_Breast_Cancer_CMS125v13.zip
get SHIW_NOV_HbA1c_CMS122v13.zip
get SHIW_NOV_High_Blood_Pressure_CMS165v13.zip
get SHIW_NOV_Screening_for_Depression_CMS2v14.zip


cd ..
cd GIP

get GIP_NOV_Breast_Cancer_CMS125v13.zip
get GIP_NOV_HbA1c_CMS122v13.zip
get GIP_NOV_High_Blood_Pressure_CMS165v13.zip
get GIP_NOV_Screening_for_Depression_CMS2v14.zip




----------------------------------------------------------ACO SFTP-------------------------------------------------------




sftp -oPort=22 paddy.g@ftp.spectramd.com

pwd ::: q7Y97P0xIHr2c




p360_bsmh_pattyg
FloatShouldTurtle933?


cd 'GI Physicians'

put GIP_NOV_Breast_Cancer_CMS125v13.zip
put GIP_NOV_Screening_for_Depression_CMS2v14.zip



cd ..
cd 'First Care Medical'

put FCM_NOV_HbA1c_CMS122v13.zip
put FCM_NOV_Breast_Cancer_CMS125v13.zip
put FCM_NOV_High_Blood_Pressure_CMS165v13.zip
put FCM_NOV_Screening_for_Depression_CMS2v14.zip


cd ..
cd 'Midwest Infectious Disease'

put MID_NOV_Screening_for_Depression_CMS2v14.zip
put MID_NOV_High_Blood_Pressure_CMS165v13.zip
put MID_NOV_Breast_Cancer_CMS125v13.zip

cd ..
cd 'Stayhome Iwill'

put SHIW_NOV_HbA1c_CMS122v13.zip
put SHIW_NOV_High_Blood_Pressure_CMS165v13.zip
put SHIW_NOV_Breast_Cancer_CMS125v13.zip
put SHIW_NOV_Screening_for_Depression_CMS2v14.zip








cd..
cd 'GI Physicians'
ls -lrth
cd ..

cd 'First Care Medical'
ls -lrth
cd ..

cd 'Midwest Infectious Disease'
ls -lrth
cd ..

cd 'Stayhome Iwill'
ls -lrth



-------------------------------------------------------------------------------------------------------------------------


Hi Paddy,

       I have generated QRDA-1 for April 2026 and uploaded in below mentioned SFTP.  Please check. Kindly let us know if any concerns.


Date Range :: 01/05/2026 - 31/05/2026

SFTP credentials:-

sftp -oPort=22 paddy.g@ftp.spectramd.com

SFTP paths:-

FCM :- /First Care Medical

MID :- /Midwest Infectious Disease

SHIW :- /Stayhome Iwill

GIP:-  /GI Physicians
 
Thanks,
Gobala Krishnan


-----------------------------------------------------------------------------------------------------------------------

sftp -oPort=8444  glenwood@ftp.glaceemr.com:Temp/Kalai/MIPS_QRDA/ACO
Gl3nW00d@$3curE$FTP!


rm SMP_HbA1c_CMS122v13.zip
rm SMP_Brest_cancer_CMS125v13.zip
rm SMP_High_Blood_Pressure_CMS165v13.zip
rm SMP_Screening_for_Depression_CMS2v14.zip
 
rm GIP_Screening_for_Depression_CMS2v14.zip
rm GIP_High_Blood_Pressure_CMS165v13.zip
rm GIP_HbA1c_CMS122v13.zip
rm GIP_Breast_Cancer_CMS125v13.zip
 
rm SHIW_HbA1c_CMS122v13.zip
rm SHIW_High_Blood_Pressure_CMS165v13.zip
rm SHIW_Screening_for_Depression_CMS2v14.zip
rm SHIW_Breast_Cancer_CMS125v13.zip
 
rm MID_High_Blood_Pressure_CMS165v13.zip
rm MID_HbA1c_CMS122v13.zip
rm MID_Breast_Cancer_CMS125v13.zip
rm MID_Screening_for_Depression_CMS2v14.zip
 
rm FCM_HbA1c_CMS122v13.zip
rm FCM_Breast_Cancer_CMS125v13.zip
rm FCM_High_Blood_Pressure_CMS165v13.zip
rm FCM_Screening_for_Depression_CMS2v14.zip

--------------------




case #244030	Question about Glenwood EMR CPT II codes/PRIMEMD(CLINIC)
