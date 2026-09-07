# Graph_Answerability_evaluation
This repository is created to support the paper for the ISWC conference

Here you can find the complete SPARQL queries used in the evaluation, together with a candidate SHACL shape derived from the Level C answerability test. The excerpts presented in the main text illustrate the graph patterns under investigation, while the full queries support reproducibility and reuse of the workflow. The SHACL example shows how a recurring structural requirement identified through SPARQL testing can be formalised as a validation warning.

1. "Level C" Entries

```sparql
PREFIX rdf: <http://www.w3.org/1999/02/22-rdf-syntax-ns#>
PREFIX rdfs: <http://www.w3.org/2000/01/rdf-schema#>
PREFIX crm: <http://www.cidoc-crm.org/cidoc-crm/>
 
SELECT DISTINCT
  ?subject
  ?subjectLabel
  ?parentStatus
  ?conditionStatus
  ?regionLabel
WHERE {
  ?subject <https://sacred_space.com/custom/has_type>
             <https://sacred_space.com/resource/hertziana/type/level_c> .
 
  OPTIONAL {
	?subject rdfs:label ?subjectLabel .
  }
 
  OPTIONAL {
	?subject crm:P46i_forms_part_of ?parentObject .
	?parentObject <https://sacred_space.com/custom/has_type>
      <https://sacred_space.com/resource/hertziana/type/level_b> .
  }
 
  OPTIONAL {
	?removal a crm:E80_Part_Removal ;
         	crm:P113_removed ?subject .
 
	FILTER NOT EXISTS {
  	?removal crm:P112_diminished [] .
	}
  }
 
  OPTIONAL {
	?subject crm:P44_has_condition ?condition .
	?condition a crm:E3_Condition_State ;
           	rdfs:label ?conditionLabel .
 
	FILTER (STR(?conditionLabel) IN (
  	"rifunzionalizzato",
  	"decontestualizzato",
  	"perduto"
	))
  }
 
  OPTIONAL {
	?subject
      (^<https://sacred_space.com/custom/includes_level_c>)?/
      (^<https://sacred_space.com/custom/includes>|crm:P46i_forms_part_of)*/
  	crm:P53_has_former_or_current_location/
  	crm:P89_falls_within+ ?region .
 
	?region crm:P2_has_type
      <https://sacred_space.com/resource/hertziana/type/region> .
 
	OPTIONAL {
  	?region rdfs:label ?regionLabel .
	}
  }
 
  BIND(
	IF(BOUND(?parentObject),
  	"has Level B parent",
  	IF(BOUND(?removal),
    	"original object unknown",
    	"no Level B parent recorded"
  	)
	) AS ?parentStatus
  )
 
  BIND(
	IF(BOUND(?conditionLabel),
  	STR(?conditionLabel),
  	"no condition recorded"
	) AS ?conditionStatus
  )
}
ORDER BY ?parentStatus ?conditionStatus ?regionLabel ?subjectLabel
```
1A. Candidate SHACL Shape for Level C

The candidate shape below identifies Level C components for which the graph records neither a Level B parent object, a condition state, nor an explicit part-removal pattern. It uses warning severity because the absence of these patterns indicates a need for curatorial review rather than an unequivocally invalid record.

```shacl
@prefix sh:    <http://www.w3.org/ns/shacl#> .
@prefix crm:   <http://www.cidoc-crm.org/cidoc-crm/> .
@prefix mss:   <https://sacred-space.com/custom/> .
@prefix msst:  <https://sacred-space.com/resource/hertziana/type/> .
@prefix msssh: <https://sacred-space.com/shapes/> .

msssh:LevelCAnswerabilityShape
    a sh:NodeShape ;
    sh:targetSubjectsOf mss:has_type ;
    sh:severity sh:Warning ;
    sh:sparql [
        a sh:SPARQLConstraint ;
        sh:message
          "No Level B parent, condition state, or unknown-object pattern is recorded." ;
        sh:select """
          PREFIX crm:
            <http://www.cidoc-crm.org/cidoc-crm/>
          PREFIX mss:
            <https://sacred-space.com/custom/>
          PREFIX msst:
            <https://sacred-space.com/resource/hertziana/type/>

          SELECT $this
          WHERE {
            $this mss:has_type msst:level_c .

            FILTER NOT EXISTS {
              $this crm:P46i_forms_part_of ?object .
              ?object mss:has_type msst:level_b .
            }

            FILTER NOT EXISTS {
              $this crm:P44_has_condition ?condition .
            }

            FILTER NOT EXISTS {
              ?removal
                  a crm:E80_Part_Removal ;
                  crm:P113_removed $this .

              FILTER NOT EXISTS {
                ?removal crm:P112_diminished [] .
              }
            }
      
```

2. Cross-Pattern Queryability with Iconclass
```sparql
PREFIX rdfs: <http://www.w3.org/2000/01/rdf-schema#>
PREFIX skos: <http://www.w3.org/2004/02/skos/core#>
PREFIX crm: <http://www.cidoc-crm.org/cidoc-crm/>

SELECT ?iconclass ?iconclassLabel
       (COUNT(DISTINCT ?subject) AS ?n_occurrences)
       (COUNT(DISTINCT ?region) AS ?n_regions)
       (COUNT(DISTINCT ?material) AS ?n_materials)
       (GROUP_CONCAT(DISTINCT ?regionLabel; separator="; ") AS ?regions)
       (GROUP_CONCAT(DISTINCT ?materialLabel; separator="; ") AS ?materials)
WHERE {
  ?subject crm:P128_carries/crm:P138_represents ?iconclass .

  ?iconclass skos:prefLabel ?iconclassLabel .
  FILTER (lang(?iconclassLabel) = "it")

  OPTIONAL {
    ?subject crm:P45_consists_of ?material .
    OPTIONAL {
      ?material rdfs:label ?materialLabel .
    }
  }

  OPTIONAL {
    ?subject
      (^<https://sacred_space.com/custom/includes_level_c>)?/
      (^<https://sacred_space.com/custom/includes>|crm:P46i_forms_part_of)*/
      crm:P53_has_former_or_current_location/
      crm:P89_falls_within+ ?region .

    ?region crm:P2_has_type
      <https://sacred_space.com/resource/hertziana/type/region> .

    OPTIONAL {
      ?region rdfs:label ?regionLabel .
    }
  }
}
GROUP BY ?iconclass ?iconclassLabel
HAVING (
  COUNT(DISTINCT ?subject) > 1 &&
  (COUNT(DISTINCT ?region) > 1 || COUNT(DISTINCT ?material) > 1)
)
ORDER BY DESC(?n_regions) DESC(?n_materials) DESC(?n_occurrences) ?iconclassLabel
```




