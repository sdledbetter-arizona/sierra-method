---
template:
  id: https://www.modelware.io/sierra/operational-analysis/alerts
  name: "Alerts"
  rank: 0
  expose:
    - kind: compose
  params:
    - id: ontology
      type: iri
      defaultValue: ${context.ontology}
      required: true
---
# Alerts

Define a mission alert, its source, target response owner, affected item, severity, and required acknowledgement window.

> Read-only context: this pattern reuses the existing Fire Force entity, stakeholder, and item definitions from the ontology, so the source and target context are treated as already-known operational context rather than a separate editable table.

```table-editor
---
columns: { this: { label: "Alert" } }
---
@prefix sh: <http://www.w3.org/ns/shacl#> .
@prefix dash: <http://datashapes.org/dash#> .
@prefix base: <https://www.modelware.io/sierra/base#> .
@prefix entity: <https://www.modelware.io/sierra/entity#> .
@prefix stakeholder: <https://www.modelware.io/sierra/stakeholder#> .
@prefix alert: <https://www.modelware.io/sierra/alert#> .

alert:AlertShape
    a sh:NodeShape ;
    sh:targetClass alert:Alert ;
    sh:property [
        sh:path alert:raisedBy ;
        sh:name "Raised By" ;
        sh:class entity:Entity ;
        sh:minCount 1 ;
        sh:maxCount 1 ;
    ] ;
    sh:property [
        sh:path alert:addressedTo ;
        sh:name "Target" ;
        sh:class stakeholder:Stakeholder ;
        sh:minCount 1 ;
        sh:maxCount 1 ;
    ] ;
    sh:property [
        sh:path alert:affects ;
        sh:name "Affected Item" ;
        sh:class base:Item ;
        sh:minCount 1 ;
        sh:maxCount 1 ;
    ] ;
    sh:property [
        sh:path alert:severity ;
        sh:name "Severity" ;
        sh:maxCount 1 ;
    ] ;
    sh:property [
        sh:path alert:acknowledgementDeadline ;
        sh:name "Ack Deadline (s)" ;
        sh:maxCount 1 ;
    ] ;
    sh:property [
        sh:path base:description ;
        sh:name "Description" ;
        dash:editor dash:TextAreaEditor ;
        sh:maxCount 1 ;
    ] ;
    sh:sparql [
        sh:message "Critical alerts must assign a target stakeholder and an acknowledgement deadline so the response can be tracked." ;
        sh:select """
            SELECT $this WHERE {
                $this a alert:Alert ;
                    alert:severity "Critical" .
                OPTIONAL { $this alert:addressedTo ?target . }
                OPTIONAL { $this alert:acknowledgementDeadline ?deadline . }
                FILTER (!BOUND(?target) || !BOUND(?deadline))
            }
        """ ;
    ] ;
    .
```

