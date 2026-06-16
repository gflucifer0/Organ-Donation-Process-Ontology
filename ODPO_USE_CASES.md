# Use Cases - Organ Donation Process Ontology (ODPO)

This document describes three use cases of the ODPO ontology (1. Educational, 2. Clinical, 3. Informative), demonstrating how SPARQL queries and automatic reasoning can provide solutions to real-world problems in the context of organ donation in Italy, both on a professional and non-professional level.

---

## Use Case 1: Verification of Brain Death Assessment Completeness

### Type and Objective
**Type**: Educational  
**Objective**: Verify that all mandatory tests and authorisations for brain death assessment have been correctly executed and recorded, ensuring compliance with CNT (Centro Nazionale Trapianti) guidelines.

### Clinical Scenario
A deceased patient undergoes brain death assessment. According to the Italian protocol, the assessment requires the execution of three groups of mandatory tests, each performed twice during the observation period:

1. **Apnea Test** (two measurements: `ApneaTest01` and `ApneaTest02`)
2. **Brainstem Reflex Test** (two measurements: `BrainstemReflexTest01` and `BrainstemReflexTest02`)
3. **Electroencephalogram (EEG)** (two measurements: `EEG01` and `EEG02`)

Moreover, the final **DiagnosisOfBrainDeath** must be **issued and provided by a CollegioMedico**, modelled in the ontology via the properties `issuedBy` and `isProvidedBy`.

In the case under examination, the system records only two tests (`ApneaTest01` and `BrainstemReflexTest01`), missing the confirmation measurements and the EEGs. The ontology must identify this incompleteness and signal that the diagnosis process is invalid or at least not complete.

### Logical Query/SPARQL

#### Approach 1: SPARQL Query for Completeness Verification

The following SPARQL query checks, for each instance of `odpo:DiagnosisOfBrainDeath`, which mandatory components are missing. It leverages the ODPO classes `ApneaTest01`, `ApneaTest02`, `BrainstemReflexTest01`, `BrainstemReflexTest02`, `EEG01`, `EEG02`, and the relation with `CollegioMedico` defined in the ontology.

```sparql
PREFIX odpo: <https://gflucifer0.github.io/Organ-Donation-Process-Ontology/ODPO.ttl#>
PREFIX rdf:  <http://www.w3.org/1999/02/22-rdf-syntax-ns#>
PREFIX rdfs: <http://www.w3.org/2000/01/rdf-schema#>

# Verify if a DiagnosisOfBrainDeath has all mandatory tests and authorisation
SELECT ?diagnosis ?missingComponent ?componentType
WHERE {
  ?diagnosis rdf:type odpo:DiagnosisOfBrainDeath .

  {
    # Verify presence of ApneaTest01
    FILTER NOT EXISTS {
      ?apneaTest01 rdf:type odpo:ApneaTest01 ;
                   odpo:confirms ?diagnosis .
    }
    BIND("ApneaTest01" AS ?missingComponent)
    BIND("ApneaTest" AS ?componentType)
  }
  UNION
  {
    # Verify presence of ApneaTest02
    FILTER NOT EXISTS {
      ?apneaTest02 rdf:type odpo:ApneaTest02 ;
                   odpo:confirms ?diagnosis .
    }
    BIND("ApneaTest02" AS ?missingComponent)
    BIND("ApneaTest" AS ?componentType)
  }
  UNION
  {
    # Verify presence of BrainstemReflexTest01
    FILTER NOT EXISTS {
      ?brainstemTest01 rdf:type odpo:BrainstemReflexTest01 ;
                       odpo:confirms ?diagnosis .
    }
    BIND("BrainstemReflexTest01" AS ?missingComponent)
    BIND("BrainstemReflexTest" AS ?componentType)
  }
  UNION
  {
    # Verify presence of BrainstemReflexTest02
    FILTER NOT EXISTS {
      ?brainstemTest02 rdf:type odpo:BrainstemReflexTest02 ;
                       odpo:confirms ?diagnosis .
    }
    BIND("BrainstemReflexTest02" AS ?missingComponent)
    BIND("BrainstemReflexTest" AS ?componentType)
  }
  UNION
  {
    # Verify presence of EEG01
    FILTER NOT EXISTS {
      ?eeg01 rdf:type odpo:EEG01 ;
             odpo:confirms ?diagnosis .
    }
    BIND("EEG01" AS ?missingComponent)
    BIND("EEG" AS ?componentType)
  }
  UNION
  {
    # Verify presence of EEG02
    FILTER NOT EXISTS {
      ?eeg02 rdf:type odpo:EEG02 ;
             odpo:confirms ?diagnosis .
    }
    BIND("EEG02" AS ?missingComponent)
    BIND("EEG" AS ?componentType)
  }
  UNION
  {
    # Verify presence of CollegioMedico (issuing and providing authority)
    FILTER NOT EXISTS {
      ?diagnosis odpo:isProvidedBy ?collegio .
      ?diagnosis odpo:issuedBy ?collegio .
      ?collegio rdf:type odpo:CollegioMedico .
    }
    BIND("CollegioMedicoAuthorization" AS ?missingComponent)
    BIND("Authorization" AS ?componentType)
  }
}
```

This query returns, for each diagnosis, the list of missing components. In a real system, the result can be used to provide feedback to clinicians and to block the completion of the process.

#### Approach 2: OWL Reasoning (Protégé/HermiT)

The ontology can be extended with a derived class `odpo:CompleteBrainDeathDiagnosis`, defined as an intersection of `DiagnosisOfBrainDeath` and the required restrictions on tests and CollegioMedico. The pattern fits the current ODPO modelling of tests (`ApneaTest01`, `ApneaTest02`, `BrainstemReflexTest01`, `BrainstemReflexTest02`, `EEG01`, `EEG02`) and of the relation with `CollegioMedico` through `isProvidedBy`.

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
        [ rdf:type owl:Restriction ;
          owl:onProperty [ owl:inverseOf odpo:confirms ] ;
          owl:someValuesFrom odpo:EEG01
        ]
        [ rdf:type owl:Restriction ;
          owl:onProperty [ owl:inverseOf odpo:confirms ] ;
          owl:someValuesFrom odpo:EEG02
        ]
      )
    ] .
```

With a reasoner (e.g., HermiT) activated in Protégé, any instance of `DiagnosisOfBrainDeath` that satisfies all these conditions will be inferred as an instance of `CompleteBrainDeathDiagnosis`. Diagnoses missing one or more components will not be classified as such.

### Expected Result

**Incomplete Scenario** (missing tests or CollegioMedico authorisation):
- The SPARQL query returns at least one row for the given diagnosis, explicitly indicating the missing component (e.g., `"ApneaTest02"`, `"EEG01"`, or `"CollegioMedicoAuthorization"`).
- The OWL reasoner does **not** classify the diagnosis as `CompleteBrainDeathDiagnosis`.
- The system signals: **"Brain death assessment process incomplete: mandatory components missing"**.

**Complete Scenario** (all tests and authorisation present):
- The SPARQL query returns no results for that diagnosis (no missing components).
- The OWL reasoner classifies the diagnosis as `CompleteBrainDeathDiagnosis`.
- The system confirms: **"Brain death assessment complete and valid"**.

This use case is intended to support medical staff training and quality control, by showing exactly which components of the protocol are missing in a given case.

### Examples

1. **Training in a university hospital ICU**  
   In a large Italian ICU that regularly reports potential donors to the national transplant network, ODPO is loaded into a teaching instance of the local clinical data warehouse. Residents simulate several brain death assessments using de-identified historical cases. For one case, the SPARQL query immediately highlights that the second EEG and the second brainstem reflex test are missing, even though the observation time is correctly recorded. The supervising intensivist uses the result to discuss with trainees how strict adherence to the national protocol protects both patients and the trust in the donation system.

2. **Quality audit after a national awareness campaign**  
   Following a national “Dai voce al tuo Sì” awareness campaign, an increase in potential donors is observed. The hospital’s transplant coordination team runs the completeness query on all diagnoses issued in the last six months. Two diagnoses are flagged as incomplete because no `CollegioMedico` instance is linked via `issuedBy` and `isProvidedBy`. The cases are traced back to legacy records migrated from an old system, and the missing links are corrected so that future reports to the Sistema Informativo Trapianti can rely on structurally complete data.

---

## Use Case 2: Automatic Exclusion of a Donor Due to Absolute Contraindication

### Type and Objective
**Type**: Clinical  
**Objective**: Automatically identify unsuitable donors based on absolute contraindications (e.g., HIV infection), using OWL reasoning and SPARQL queries to ensure safety and regulatory compliance.

### Clinical Scenario
A deceased potential donor presents a diagnosis of **ongoing HIV infection** (`HIVInfection`). According to CNT guidelines and the modelling in ODPO, an ongoing HIV infection is an instance of `AbsoluteContraindication` that completely excludes the donor from the donation process, regardless of the organ considered.

In the ontology, `NotSuitableDonor` is defined as a class equivalent to the intersection of `PotentialOrganDonor` and the existence of at least one `contraindicatedBy` relation to an `AbsoluteContraindication`. The system must:

1. Detect absolute contraindications linked to a potential donor via `odpo:contraindicatedBy`.
2. Classify the donor as `odpo:NotSuitableDonor` through automatic reasoning.
3. Preventively block any `OrganProcurement` procedure involving that donor.

### Logical Query/SPARQL

#### Approach 1: SPARQL Query for Explicit Identification

```sparql
PREFIX odpo: <https://gflucifer0.github.io/Organ-Donation-Process-Ontology/ODPO.ttl#>
PREFIX rdf:  <http://www.w3.org/1999/02/22-rdf-syntax-ns#>
PREFIX rdfs: <http://www.w3.org/2000/01/rdf-schema#>

# Identify potential donors with absolute contraindications
SELECT ?donor ?contraindication ?contraindicationLabel
WHERE {
  ?donor rdf:type odpo:PotentialOrganDonor .
  ?donor odpo:contraindicatedBy ?contraindication .
  ?contraindication rdf:type odpo:AbsoluteContraindication .

  OPTIONAL { ?contraindication rdfs:label ?contraindicationLabel . }
}
```

This query returns all potential donors who are linked to at least one absolute contraindication, together with the label of the contraindication when available.

#### Approach 2: OWL Reasoning (Automatic Classification)

In ODPO, the class `NotSuitableDonor` is defined as follows (simplified excerpt):

```turtle
odpo:NotSuitableDonor
  rdf:type owl:Class ;
  owl:equivalentClass [
    owl:intersectionOf (
      odpo:PotentialOrganDonor
      [ rdf:type owl:Restriction ;
        owl:onProperty odpo:contraindicatedBy ;
        owl:someValuesFrom odpo:AbsoluteContraindication
      ]
    ) ;
    rdf:type owl:Class
  ] ;
  rdfs:subClassOf odpo:PotentialOrganDonor .
```

A donor with an active HIV infection can be modelled as follows:

```turtle
odpo:donor_001
  rdf:type odpo:PotentialOrganDonor ;
  odpo:contraindicatedBy odpo:HIVInfection .

odpo:HIVInfection
  rdf:type odpo:AbsoluteContraindication ,
           odpo:MedicalCondition .
```

With a reasoner enabled, `odpo:donor_001` will automatically be inferred as an instance of `odpo:NotSuitableDonor`.

### Expected Result

**With Active OWL Reasoner** (Protégé/HermiT/Pellet):
- The reasoner classifies `odpo:donor_001` as an instance of `odpo:NotSuitableDonor`.
- The SPARQL query above returns a row such as:
?donor = odpo:donor_001
?contraindication = odpo:HIVInfection
?contraindicationLabel = "HIV Infection"

text

**System Behaviour**:
1. **Preventive Block**: any attempt to create an `OrganProcurement` event involving `odpo:donor_001` is blocked.
2. **Automatic Notification**: the system raises an alert such as **"Donor excluded: absolute contraindication (HIV)"**.
3. **Traceability**: the exclusion decision can be logged and audited as part of the `OrganDonationProcess`.

This use case shows how ODPO can support clinical decision-making and safety checks by combining OWL reasoning with SPARQL-based inspection.

### Examples

1. **Screening donors in a high-volume region**  
 In a region with a high rate of donation and transplantation activity, the local coordination centre uses ODPO-based rules to screen all potential donors submitted by ICUs. When a case with a recent diagnosis of an aggressive malignancy is entered, the clinical documentation is mapped to an instance of `AbsoluteContraindication`. The reasoner immediately classifies the deceased as `NotSuitableDonor`, and the transplant coordinator can explain to the family that, despite the generosity of their choice, donation is not clinically safe in this specific case.

2. **Harmonising practice across multiple hospitals**  
 After a national audit highlights variability in how certain infections are treated as contraindications, an ODPO-driven decision support module is deployed in several hospitals. For a borderline case of sepsis, the infectious disease specialist initially hesitates. By modelling the condition as a `RelativeContraindication` instead of an `AbsoluteContraindication`, the system allows the multidisciplinary team to record additional tests and decide case by case, while still automatically blocking donors for clearly defined absolute conditions such as specific untreatable infections.

---

## Use Case 3: Traceability of a Safe Organ from Suitable Donor to Recipient

### Type and Objective
**Type**: Informative  
**Objective**: Provide complete and transparent traceability of an organ's path from the moment of donation until transplantation, demonstrating the safety and rigor of the process for informational and public transparency purposes.

### Clinical Scenario
A liver is retrieved from a suitable donor (`OrganDonor`) and successfully transplanted into a recipient (`Recipient`). Using the ODPO ontology, we want to reconstruct the entire chain of custody:

1. **Donor identification** and suitability verification, using the distinction between `PotentialOrganDonor`, `NotSuitableDonor` and `OrganDonor`.
2. **Organ procurement** (`OrganProcurement`) linked to the organ through `resultsFrom` and `hasProcurement`.
3. **Transplantation** (`Transplant`) involving the organ (`involves`) and the recipient (`isTransplantedInto`).
4. **Safety verification**: absence of absolute contraindications connected via `contraindicatedBy`.

This path must be queryable to demonstrate that the organ comes from a safe and compliant process, according to the modelling of ODPO.

### Logical Query/SPARQL

```sparql
PREFIX odpo: <https://gflucifer0.github.io/Organ-Donation-Process-Ontology/ODPO.ttl#>
PREFIX rdf:  <http://www.w3.org/1999/02/22-rdf-syntax-ns#>
PREFIX rdfs: <http://www.w3.org/2000/01/rdf-schema#>
PREFIX xsd:  <http://www.w3.org/2001/XMLSchema#>

# Trace the complete path of an organ from donor to recipient
SELECT ?organ ?organType ?donor ?procurement ?procurementDate 
     ?transplant ?transplantDate ?recipient ?safetyStatus
WHERE {
# Organ
?organ rdf:type odpo:Organ .
OPTIONAL { ?organ rdfs:label ?organType . }

# Donor
?organ odpo:isDonatedFrom ?donor .
?donor rdf:type odpo:OrganDonor .

# Verify that the donor is NOT a NotSuitableDonor (safety precondition)
FILTER NOT EXISTS {
  ?donor rdf:type odpo:NotSuitableDonor .
}

# Procurement
?organ odpo:resultsFrom ?procurement .
?procurement rdf:type odpo:OrganProcurement .
OPTIONAL { 
  ?procurement odpo:procurementDate ?procurementDate .
}

# Transplant and recipient
?organ odpo:isTransplantedInto ?recipient .
?recipient rdf:type odpo:Recipient .
?transplant rdf:type odpo:Transplant .
?transplant odpo:involves ?organ .
OPTIONAL {
  ?transplant odpo:transplantDate ?transplantDate .
}

# Safety verification: no absolute contraindications for the donor
BIND(
  IF(
    NOT EXISTS { ?donor odpo:contraindicatedBy ?absContra . 
                 ?absContra rdf:type odpo:AbsoluteContraindication . },
    "SAFE - No absolute contraindications",
    "UNSAFE - Absolute contraindication present"
  ) AS ?safetyStatus
)
}
ORDER BY ?transplantDate
```

This query follows the links defined in ODPO (`isDonatedFrom`, `resultsFrom`, `isTransplantedInto`, `involves`, `contraindicatedBy`) to retrieve a complete picture of the organ’s journey.

#### Alternative Query: Traceability for a Specific Organ Type

```sparql
PREFIX odpo: <https://gflucifer0.github.io/Organ-Donation-Process-Ontology/ODPO.ttl#>
PREFIX rdf:  <http://www.w3.org/1999/02/22-rdf-syntax-ns#>
PREFIX rdfs: <http://www.w3.org/2000/01/rdf-schema#>
PREFIX xsd:  <http://www.w3.org/2001/XMLSchema#>

# Trace a specific organ type (e.g., liver) from donor to recipient
SELECT ?organ ?donor ?procurementDate ?transplantOutcome
WHERE {
?organ rdf:type odpo:Liver .   # or odpo:Kidney, odpo:Heart, etc.

# Donor and suitability
?organ odpo:isDonatedFrom ?donor .
?donor rdf:type odpo:OrganDonor .
FILTER NOT EXISTS { ?donor rdf:type odpo:NotSuitableDonor . }

# Procurement
?organ odpo:resultsFrom ?procurement .
?procurement rdf:type odpo:OrganProcurement .
OPTIONAL {
  ?procurement odpo:procurementDate ?procurementDate .
}

# Recipient and transplant outcome
?organ odpo:isTransplantedInto ?recipient .
?recipient rdf:type odpo:Recipient .

?transplant rdf:type odpo:Transplant .
?transplant odpo:involves ?organ .
OPTIONAL {
  ?transplant odpo:transplantOutcome ?transplantOutcome .
}
}
```

### Expected Result

For a successfully transplanted safe organ, the first query may return a result such as:

```text
?organ           = odpo:liver_2026_045
?organType       = "Liver"
?donor           = odpo:donor_marco_rossi
?procurement     = odpo:procurement_2026_045
?procurementDate = "2026-02-20T10:15:00"^^xsd:dateTime
?transplant      = odpo:transplant_2026_045
?transplantDate  = "2026-02-20T14:30:00"^^xsd:dateTime
?recipient       = odpo:recipient_R_TRAPI_221
?safetyStatus    = "SAFE - No absolute contraindications"
```

This output can be rendered by an application as a human-readable traceability report, thereby supporting:

- **Public transparency**: citizens can verify the safety and traceability of organs used in transplantation.
- **Accountability**: complete traceability for audit and regulatory purposes.
- **Scientific dissemination**: demonstration of the rigor of the Italian transplant system.
- **Trust**: increased public trust in the organ donation and transplantation process.

The same query patterns can be adapted to generate aggregated statistical reports (e.g., "How many safe liver transplants were performed in 2026?").

### Examples

1. **Storytelling for a public awareness website**  
 A regional health authority develops a public web page that explains how donation works, echoing national campaigns that emphasise that one “yes” can save multiple lives. For each anonymised story shown on the site, the back-end system uses the ODPO traceability query to generate a narrative: identification of a potential donor, confirmation of brain death by the `CollegioMedico`, consent registration in the national information system, procurement, and successful transplant. Only non-identifiable labels are displayed, but the underlying linked data ensure that every step is consistent with the clinical workflow.

2. **Internal audit using the national transplant information system**  
 As part of a periodic audit, a hospital extracts a subset of donation and transplant data from the Sistema Informativo Trapianti and maps it to the ODPO ontology. By running the traceability queries for all heart transplants performed in a given year, the quality team checks that each organ has an associated `OrganProcurement`, that the donor is not classified as `NotSuitableDonor`, and that a `Transplant` event with a documented outcome exists. Any missing link is flagged and reconciled with the source systems, strengthening both data quality and the hospital’s ability to demonstrate compliance with national standards.

---

**Version**: 1.0.0 
**Date**: 06/2026  
**Author**: Giacomo F. Lucifero, Alma Mater Studiorum - University of Bologna