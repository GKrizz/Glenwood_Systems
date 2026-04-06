### 🔹 What we are handling now

1. **Vitals-based CPT II automation (BP & BMI)**

   * Systolic & Diastolic BP values are captured from vitals.
   * Based on the latest values:

     * Systolic → 3074F / 3075F / 3077F
     * Diastolic → 3078F / 3079F / 3080F
   * Logic is implemented in backend (`ChargesServicesImpl → getVitals()`).

2. **Lab-based CPT II automation (HbA1c)**

   * Triggered when Hemoglobin lab results (LOINC codes) are received.
   * CPT II generated based on result:

     * <7 → 3044F
     * 7–8 → 3051F
     * 8–9 → 3052F
     * > 9 → 3046F

3. **Other HEDIS measures supported**

   * BMI (G8420, G8417, etc.)
   * Depression screening (G8431, G8510, etc.)
   * Medication review (G8427/G8428)
   * Advance care planning (1123F/1124F)

4. **HEDIS Configuration Driven**

   * CPT generation happens only if:

     * Reporting year is active (`hedis_configuration`)
     * Provider is mapped to measures (`hedis_measure_provider_configuration`)

---

### 🔹 How we are maintaining it

1. **Centralized Logic**

   * All CPT II generation is handled in:

     * `ChargesServicesImpl.saveServicesBasedOnHedisMeasuresConfig()`

2. **Duplicate Handling**

   * Existing CPTs are checked using `getExistingServiceMap()`
   * System:

     * Updates if CPT changed
     * Deletes if no longer applicable
     * Inserts only if new

3. **Data Source Driven**

   * CPT codes are derived from:

     * Vitals (BP, BMI)
     * Lab results (HbA1c)
     * Screenings / Plan data

4. **Stored in Billing Layer**

   * Final CPTs are saved in `service_detail`
   * Used for:

     * Billing
     * MIPS/HEDIS reporting
     * CDA export

---
