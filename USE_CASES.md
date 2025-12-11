# Use Cases - Organ Donation Process Ontology (ODPO)

This document describes three use cases of the ODPO ontology (1. Educational, 2. Clinical, 3. Informative), demonstrating how SPARQL queries and automatic reasoning can provide solutions to real-world problems in the context of organ donation in Italy both on a professional and unprofessional level.

---

## Use Case 1: Verification of Brain Death Assessment Completeness

### Title and Objective
**Type**: Educational  
**Objective**: Verify that all mandatory tests for brain death assessment have been correctly executed, ensuring compliance with CNT (Centro Nazionale Trapianti) guidelines.

### Clinical Scenario
A patient in an irreversible coma undergoes brain death assessment. According to the Italian protocol, the assessment requires the execution of **three mandatory tests**:
1. **Apnea Test** (two measurements: ApneaTest01 and ApneaTest02)
2. **Brainstem Reflex Test** (two measurements: BrainstemReflexTest01 and BrainstemReflexTest02)
3. **Electroencephalogram (EEG)** (two measurements: EEG01 and EEG02)

In the case under examination, the system records only two tests (ApneaTest01 and BrainstemReflexTest01), missing the confirmation measurements. The ontology must identify this incompleteness and signal that the diagnosis process is invalid.

### Logical Query/SPARQL

#### Approach 1: SPARQL Query for Completeness Verification

```sparql
PREFIX odpo: <http://www.semanticweb.org/giacomo_lucifero/ontologies/2025/10/organ-donation-process-ontology#>
PREFIX rdf: <http://www.w3.org/1999/02/22-rdf-syntax-ns#>
PREFIX rdfs: <http://www.w3.org/2000/01/rdf-schema#>

# Verify if a DiagnosisOfBrainDeath has all mandatory tests
SELECT ?diagnosis ?missingTest ?testType
WHERE {
    ?diagnosis rdf:type odpo:DiagnosisOfBrainDeath .
    
    # Mandatory tests according to CNT protocol
    {
        # Verify presence of ApneaTest01
        FILTER NOT EXISTS {
            ?apneaTest01 rdf:type odpo:ApneaTest01 ;
                         odpo:confirms ?diagnosis .
        }
        BIND("ApneaTest01" AS ?missingTest)
        BIND("ApneaTest" AS ?testType)
    }
    UNION
    {
        # Verify presence of ApneaTest02
        FILTER NOT EXISTS {
            ?apneaTest02 rdf:type odpo:ApneaTest02 ;
                         odpo:confirms ?diagnosis .
        }
        BIND("ApneaTest02" AS ?missingTest)
        BIND("ApneaTest" AS ?testType)
    }
    UNION
    {
        # Verify presence of BrainstemReflexTest01
        FILTER NOT EXISTS {
            ?brainstemTest01 rdf:type odpo:BrainstemReflexTest01 ;
                             odpo:confirms ?diagnosis .
        }
        BIND("BrainstemReflexTest01" AS ?missingTest)
        BIND("BrainstemReflexTest" AS ?testType)
    }
    UNION
    {
        # Verify presence of BrainstemReflexTest02
        FILTER NOT EXISTS {
            ?brainstemTest02 rdf:type odpo:BrainstemReflexTest02 ;
                             odpo:confirms ?diagnosis .
        }
        BIND("BrainstemReflexTest02" AS ?missingTest)
        BIND("BrainstemReflexTest" AS ?testType)
    }
    UNION
    {
        # Verify presence of CollegioMedico (issuing authority)
        FILTER NOT EXISTS {
            ?diagnosis odpo:isProvidedBy ?collegio .
            ?collegio rdf:type odpo:CollegioMedico .
        }
        BIND("CollegioMedicoAuthorization" AS ?missingTest)
        BIND("Authorization" AS ?testType)
    }
}
```

#### Approach 2: OWL Reasoning (Protégé/HermiT)

The ontology can be extended with a restriction that defines a `CompleteBrainDeathDiagnosis` class:

```turtle
odpo:CompleteBrainDeathDiagnosis
    rdf:type owl:Class ;
    owl:equivalentClass [
        rdf:type owl:Class ;
        owl:intersectionOf (
            odpo:DiagnosisOfBrainDeath
            [ rdf:type owl:Restriction ;
              owl:onProperty odpo:isProvidedBy ;
              owl:someValuesFrom odpo:CollegioMedico
            ]
            [ rdf:type owl:Restriction ;
              owl:onProperty [ owl:inverseOf odpo:confirms ] ;
              owl:someValuesFrom odpo:ApneaTest01
            ]
            [ rdf:type owl:Restriction ;
              owl:onProperty [ owl:inverseOf odpo:confirms ] ;
              owl:someValuesFrom odpo:ApneaTest02
            ]
            [ rdf:type owl:Restriction ;
              owl:onProperty [ owl:inverseOf odpo:confirms ] ;
              owl:someValuesFrom odpo:BrainstemReflexTest01
            ]
            [ rdf:type owl:Restriction ;
              owl:onProperty [ owl:inverseOf odpo:confirms ] ;
              owl:someValuesFrom odpo:BrainstemReflexTest02
            ]
        )
    ] .
```

The reasoner will automatically classify a `DiagnosisOfBrainDeath` as `CompleteBrainDeathDiagnosis` only if it satisfies all conditions.

### Expected Result

**Incomplete Scenario** (missing tests):
- The SPARQL query returns:
  ```
  ?diagnosis = odpo:diagnosis_001
  ?missingTest = "ApneaTest02"
  ?testType = "ApneaTest"
  ```
- The OWL reasoner does **not** classify the diagnosis as `CompleteBrainDeathDiagnosis`
- The system signals: **"Brain death assessment process incomplete: missing ApneaTest02"**

**Complete Scenario** (all tests present):
- The SPARQL query returns no results (no missing tests)
- The OWL reasoner classifies the diagnosis as `CompleteBrainDeathDiagnosis`
- The system confirms: **"Brain death assessment complete and valid"**

This particular kind of use case is intended to support medical staff training by showing exactly which components are missing in the process.

---

## Use Case 2: Automatic Exclusion of a Donor Due to Absolute Contraindication

### Title and Objective
**Type**: Clinical  
**Objective**: Automatically identify unsuitable donors based on absolute contraindications (e.g., HIV infection), using automatic OWL reasoning to ensure safety and regulatory compliance.

### Clinical Scenario
A deceased potential donor presents a diagnosis of **ongoing HIV infection** (HIVInfection). According to CNT guidelines, an ongoing HIV infection is an **absolute contraindication** (`AbsoluteContraindication`) that completely excludes the donor from the donation process, regardless of the organ considered.

The system must:
1. Automatically detect the presence of HIV in the donor
2. Classify the donor as `NotSuitableDonor` through reasoning
3. Preventively block any procurement procedure

### Logical Query/SPARQL

#### Approach 1: SPARQL Query for Explicit Identification

```sparql
PREFIX odpo: <http://www.semanticweb.org/giacomo_lucifero/ontologies/2025/10/organ-donation-process-ontology#>
PREFIX rdf: <http://www.w3.org/1999/02/22-rdf-syntax-ns#>
PREFIX rdfs: <http://www.w3.org/2000/01/rdf-schema#>

# Identify donors with absolute contraindications
SELECT ?donor ?contraindication ?contraindicationType
WHERE {
    ?donor rdf:type odpo:PotentialOrganDonor .
    ?donor odpo:contraindicatedBy ?contraindication .
    ?contraindication rdf:type odpo:AbsoluteContraindication .
    
    OPTIONAL {
        ?contraindication rdfs:label ?contraindicationType .
    }
}
```

#### Approach 2: OWL Reasoning (Automatic Classification)

The ODPO ontology defines `NotSuitableDonor` as:

```turtle
odpo:NotSuitableDonor
    owl:equivalentClass [
        owl:intersectionOf (
            odpo:PotentialOrganDonor
            [ rdf:type owl:Restriction ;
              owl:onProperty odpo:contraindicatedBy ;
              owl:someValuesFrom odpo:AbsoluteContraindication
            ]
        )
    ] .
```

**Donor instance with HIV**:
```turtle
odpo:donor_001
    rdf:type odpo:PotentialOrganDonor ;
    odpo:contraindicatedBy odpo:HIVInfection .

odpo:HIVInfection
    rdf:type odpo:AbsoluteContraindication ,
             odpo:MedicalCondition .
```

### Expected Result

**With Active OWL Reasoner** (Protégé/HermiT/Pellet):
- The reasoner automatically classifies `donor_001` as an instance of `NotSuitableDonor`
- The SPARQL query returns:
  ```
  ?donor = odpo:donor_001
  ?contraindication = odpo:HIVInfection
  ?contraindicationType = "HIV Infection"
  ```

**System Behavior**:
1. **Preventive Block**: Any attempt to initiate an `OrganProcurement` for `donor_001` is blocked
2. **Automatic Notification**: The system generates an alert: **"Donor excluded: absolute contraindication (HIV)"**
3. **Traceability**: The exclusion is recorded in the process log for audit purposes

This particular kind of use is intended to be conducted and supervised by healthcare professionals in order to provide step-by-step control of the procedures implemented.
---

## Use Case 3: Traceability of a Safe Organ from Suitable Donor to Recipient

### Title and Objective
**Type**: Informative 
**Objective**: Provide complete and transparent traceability of an organ's path from the moment of donation until transplantation, demonstrating the safety and rigor of the process for informational and public transparency purposes.

### Clinical Scenario
A liver is retrieved from a suitable donor (`OrganDonor`) and successfully transplanted into a recipient. For informational and transparency purposes, it is necessary to reconstruct the entire chain of custody:

1. **Donor identification** and suitability verification
2. **Organ procurement** (`OrganProcurement`)
3. **Transplantation** (`Transplant`) to the recipient
4. **Safety verification** (no contraindications present)

This path must be queryable to demonstrate that the organ comes from a safe and compliant process.

### Logical Query/SPARQL

```sparql
PREFIX odpo: <http://www.semanticweb.org/giacomo_lucifero/ontologies/2025/10/organ-donation-process-ontology#>
PREFIX rdf: <http://www.w3.org/1999/02/22-rdf-syntax-ns#>
PREFIX rdfs: <http://www.w3.org/2000/01/rdf-schema#>
PREFIX xsd: <http://www.w3.org/2001/XMLSchema#>

# Trace the complete path of an organ from donor to recipient
SELECT ?organ ?organType ?donor ?donorStatus ?procurement ?procurementDate 
       ?transplant ?transplantDate ?recipient ?safetyStatus
WHERE {
    # Organ
    ?organ rdf:type odpo:Organ .
    OPTIONAL { ?organ rdfs:label ?organType . }
    
    # Donor
    ?organ odpo:isDonatedFrom ?donor .
    ?donor rdf:type odpo:OrganDonor .
    
    # Verify that the donor is NOT NotSuitableDonor
    FILTER NOT EXISTS {
        ?donor rdf:type odpo:NotSuitableDonor .
    }
    
    # Procurement
    ?organ odpo:resultsFrom ?procurement .
    ?procurement rdf:type odpo:OrganProcurement .
    OPTIONAL { 
        ?procurement odpo:procurementDate ?procurementDate .
    }
    
    # Transplant
    ?organ odpo:isTransplantedInto ?recipient .
    ?transplant rdf:type odpo:Transplant .
    ?transplant odpo:involves ?organ .
    OPTIONAL {
        ?transplant odpo:transplantDate ?transplantDate .
    }
    
    # Safety verification: no absolute contraindications
    BIND(
        IF(
            NOT EXISTS { ?donor odpo:contraindicatedBy ?absContra . 
                         ?absContra rdf:type odpo:AbsoluteContraindication . },
            "SAFE - No absolute contraindications",
            "UNSAFE - Contraindication present"
        ) AS ?safetyStatus
    )
}
ORDER BY ?transplantDate
```

#### Alternative Query: Traceability for Specific Organ

```sparql
PREFIX odpo: <http://www.semanticweb.org/giacomo_lucifero/ontologies/2025/10/organ-donation-process-ontology#>
PREFIX rdf: <http://www.w3.org/1999/02/22-rdf-syntax-ns#>

# Trace a specific organ (e.g., liver) from donor to recipient
SELECT ?organ ?donorName ?procurementLocation ?recipientCode ?transplantOutcome
WHERE {
    ?organ rdf:type odpo:Liver .  # or odpo:Kidney, odpo:Heart, etc.
    
    # Donor
    ?organ odpo:isDonatedFrom ?donor .
    ?donor rdf:type odpo:OrganDonor .
    OPTIONAL { ?donor rdfs:label ?donorName . }
    
    # Verify donor suitability
    FILTER NOT EXISTS {
        ?donor rdf:type odpo:NotSuitableDonor .
    }
    
    # Procurement
    ?organ odpo:resultsFrom ?procurement .
    OPTIONAL {
        ?procurement odpo:occursIn ?location .
        ?location rdfs:label ?procurementLocation .
    }
    
    # Recipient
    ?organ odpo:isTransplantedInto ?recipient .
    ?recipient rdf:type odpo:Recipient .
    OPTIONAL {
        ?recipient odpo:recipientCode ?recipientCode .
    }
    
    # Transplant outcome
    ?transplant rdf:type odpo:Transplant .
    ?transplant odpo:involves ?organ .
    OPTIONAL {
        ?transplant odpo:transplantOutcome ?transplantOutcome .
    }
}
```

### Expected Result

**Query Executed Successfully**:
```
?organ = odpo:liver_2025_045
?organType = "Liver"
?donor = odpo:donor_marco_rossi
?donorStatus = "OrganDonor (suitable)"
?procurement = odpo:procurement_2025_045
?procurementDate = "2025-02-20"^^xsd:date
?transplant = odpo:transplant_2025_045
?transplantDate = "2025-02-20"^^xsd:date
?recipient = odpo:recipient_R_TRAPI_221
?safetyStatus = "SAFE - No absolute contraindications"
```

**Informative Output** (readable format):
```
═══════════════════════════════════════════════════════════
  ORGAN TRACEABILITY: Liver (liver_2025_045)
═══════════════════════════════════════════════════════════

DONOR:
  • Identifier: donor_marco_rossi
  • Status: Suitable (verified)
  • Contraindications: None

PROCUREMENT:
  • Date: 2025-02-20
  • Center: Regional Hospital
  • Authorization: Valid consent

TRANSPLANT:
  • Date: 2025-02-20
  • Recipient: R-TRAPI-221
  • Outcome: Success

SAFETY STATUS: ✅ SAFE
  • No absolute contraindications detected
  • Process compliant with CNT guidelines
═══════════════════════════════════════════════════════════
```

**Added Value**:
- **Public Transparency**: Citizens can verify the safety of the process
- **Accountability**: Complete traceability for audit and regulatory compliance
- **Scientific Dissemination**: Demonstrates the rigor of the Italian transplant system
- **Trust**: Increases public trust in the organ donation system

**Extension**: The same query can be used to generate aggregated statistical reports (e.g., "How many safe organs were transplanted in 2025?").

---

## Conclusions

These three use cases demonstrate how the ODPO ontology supports:

1. **Training** (Case 1): Automatic verification of clinical process completeness
2. **Control** (Case 2): Automatic exclusion of unsuitable donors
3. **Transparency** (Case 3): Complete traceability for public information

---

**Version**: 1.0  
**Date**: 2025-02  
**Author**: Giacomo F. Lucifero, Alma Mater Studiorum - University of Bologna
