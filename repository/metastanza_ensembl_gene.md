## Endpoint

https://integbio.jp/togosite/sparql

## Parameters

* `id`
  * example: ENSG00000150773, ENST00000280350
  
## `is_ensg`
```javascript
({ id }) => {
  if (id.match(/^ENSG[0-9]{11}$/)) {
    return true;
  } else if (id.match(/^ENST[0-9]{11}$/)) {
    return false;
  } 
}
```

## `main`

```sparql
PREFIX rdfs: <http://www.w3.org/2000/01/rdf-schema#>
PREFIX obo: <http://purl.obolibrary.org/obo/>
PREFIX so: <http://purl.obolibrary.org/obo/so#>
PREFIX idt_ensg: <http://identifiers.org/ensembl/>
PREFIX dc: <http://purl.org/dc/elements/1.1/>
PREFIX dct: <http://purl.org/dc/terms/>
PREFIX taxon: <http://identifiers.org/taxonomy/>
PREFIX faldo: <http://biohackathon.org/resource/faldo#>
PREFIX refexo: <http://purl.jp/bio/01/refexo#>
PREFIX schema: <http://schema.org/>
PREFIX ensg: <http://rdf.ebi.ac.uk/resource/ensembl/>
PREFIX enst: <http://rdf.ebi.ac.uk/resource/ensembl.transcript/>

SELECT DISTINCT ?ensg_id ?idt_ensg ?gene_symbol ?desc ?type_label ?location ?type_gtex ?type_hpa_cell ?type_hpa_tissue
  ?begin ?end ?strand
  (GROUP_CONCAT(DISTINCT ?gtex_tissue_label; separator=", ") AS ?gtex_tissue_labels)
  (GROUP_CONCAT(DISTINCT ?hpa_tissue_label; separator=", ") AS ?hpa_tissue_labels)
  (GROUP_CONCAT(DISTINCT ?hpa_cell_label; separator=", ") AS ?hpa_cell_labels)
WHERE {
  {{#if is_ensg}}
    VALUES ?ensg { ensg:{{id}} }
  {{else}}
    VALUES ?input_enst { enst:{{id}} }
    ?input_enst so:transcribed_from ?ensg .
  {{/if}}
  BIND(URI(REPLACE(STR(?ensg), "http://rdf.ebi.ac.uk/resource/ensembl/", "http://identifiers.org/ensembl/")) AS ?idt_ensg)
  VALUES ?strand { faldo:ReverseStrandPosition faldo:ForwardStrandPosition }
  GRAPH <http://rdf.integbio.jp/dataset/togosite/ensembl> {
    ?ensg obo:RO_0002162 taxon:9606 ;
          dct:identifier ?ensg_id ;
          rdfs:label ?gene_symbol ;
          dct:description ?desc ;
          faldo:location [
            faldo:begin [
              a ?strand ;
              faldo:position ?begin
            ] ;
            faldo:end [
              faldo:position ?end
            ]
          ] ;
          so:part_of ?chr ;
          a ?type .
    #?loc rdfs:label ?location .
    FILTER(STRSTARTS(STR(?type), "http://rdf.ebi.ac.uk/terms/ensembl/"))
    BIND(REPLACE(STRAFTER(STR(?type), "http://rdf.ebi.ac.uk/terms/ensembl/"), "_", " ") as ?type_label)
    BIND(STRBEFORE(STRAFTER(STR(?chr), "http://identifiers.org/hco/"), "#") as ?location)
  }

  OPTIONAL {
    GRAPH <http://rdf.integbio.jp/dataset/togosite/refex_tissue_specific_genes_gtex_v6> {
      ?idt_ensg a ?type_gtex .
    }
  }
  OPTIONAL {
    GRAPH <http://rdf.integbio.jp/dataset/togosite/hpa_cell_specificity> {
      ?idt_ensg a ?type_hpa_cell .
    }
  }
  OPTIONAL {
    GRAPH <http://rdf.integbio.jp/dataset/togosite/hpa_tissue_specificity> {
      ?idt_ensg a ?type_hpa_tissue .
    }
  }

  OPTIONAL {
    GRAPH <http://rdf.integbio.jp/dataset/togosite/refex_tissue_specific_genes_gtex_v6> {
      ?idt_ensg refexo:isPositivelySpecificTo ?gtex_tissue .
    }
    GRAPH <http://rdf.integbio.jp/dataset/togosite/refexsample_gtex_v8_summary> {
      VALUES ?name {"cell type" "tissue"}
      ?refexs schema:additionalProperty [
        schema:name ?name ;
        schema:value ?gtex_tissue_label ;
        schema:valueReference ?gtex_tissue
      ] .
    }
  }
  OPTIONAL {
    GRAPH <http://rdf.integbio.jp/dataset/togosite/hpa_cell_specificity> {
      ?idt_ensg refexo:isPositivelySpecificTo ?hpa_cell .
    }
    GRAPH <http://rdf.integbio.jp/dataset/togosite/caloha> {
      ?hpa_cell rdfs:label ?hpa_cell_label .
    }
  }
  OPTIONAL {
    GRAPH <http://rdf.integbio.jp/dataset/togosite/hpa_tissue_specificity> {
      ?idt_ensg refexo:isPositivelySpecificTo ?hpa_tissue .
    }
    GRAPH <http://rdf.integbio.jp/dataset/togosite/caloha> {
      ?hpa_tissue rdfs:label ?hpa_tissue_label .
    }
  }
}
```

## `return`

```javascript
({ main, id }) => {
  let data = main.results.bindings[0];
  let ts_gtex = data.gtex_tissue_labels.value;
  if (ts_gtex == "") {
    if (data.type_gtex?.value)
      ts_gtex = "(Low tissue specificity)";
    else 
      ts_gtex = "N/A";
  } else {
    ts_gtex = "<ul><li>" + ts_gtex.split(", ").join("</li><li>") + "</li></ul>";
  }
  let ts_hpa = data.hpa_tissue_labels.value;
  if (ts_hpa == "") {
    if (data.type_hpa_tissue?.value)
      ts_hpa = "(Low tissue specificity)";
    else 
      ts_hpa = "N/A";
  } else {
    ts_hpa = "<ul><li>" + ts_hpa.split(", ").join("</li><li>") + "</li></ul>";
  }
  let cs_hpa = data.hpa_cell_labels.value;
  if (cs_hpa == "") {
    if (data.type_hpa_cell?.value)
      cs_hpa = "(Low cell specificity)";
    else 
      cs_hpa = "N/A";
  } else {
    cs_hpa = "<ul><li>" + cs_hpa.split(", ").join("</li><li>") + "</li></ul>";
  }

  let location = "chr" + data.location.value + ":" + data.begin.value + "-" + data.end.value;
  if (data.strand.value == "http://biohackathon.org/resource/faldo#ForwardStrandPosition") {
    location = location + " forward strand";
  } else if (data.strand.value == "http://biohackathon.org/resource/faldo#ReverseStrandPosition") {
    location = location + " reverse strand";
  }
  let objs = [{
    "Ensembl ID": data.ensg_id.value,
    "Ensembl URL": data.idt_ensg.value,
    "Gene symbol": data.gene_symbol.value,
    "Description": (data.desc?.value) ? data.desc.value : "",
    "Gene type": data.type_label.value,
    "Location": location,
    "Tissue specificity (GTEx)": ts_gtex,
    "Tissue specificity (HPA)": ts_hpa,
    "Cell specificity (HPA)": cs_hpa,
    "Expression": "<a href=\"https://gtexportal.org/home/gene/" + id + "\" target=\"_blank\">" + "View Expression at GTEx Portal</a>, " +
                  "<a href=\"https://www.proteinatlas.org/" + id + "\" target=\"_blank\">" + "View Expression at ProteinAtlas</a>"
  }];

  return objs;
};
```
