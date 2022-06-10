# staging-algorithm-tnm

[![Quality Gate Status](https://sonarcloud.io/api/project_badges/measure?project=imsweb_staging-algorithm-tnm&metric=alert_status)](https://sonarcloud.io/summary/new_code?id=imsweb_staging-algorithm-tnm)
[![integration](https://github.com/imsweb/staging-algorithm-tnm/workflows/integration/badge.svg)](https://github.com/imsweb/staging-algorithm-tnm/actions)
[![Maven Central](https://maven-badges.herokuapp.com/maven-central/com.imsweb/staging-algorithm-tnm/badge.svg)](https://maven-badges.herokuapp.com/maven-central/com.imsweb/staging-algorithm-tnm)

TNM is a widely accepted system of cancer staging. TNM stands for Tumor, Nodes, and Metastasis. T is assigned based on the extent of involvement at 
the primary tumor site, N for the extent of involvement in regional lymph nodes, and M for distant spread. Clinical TNM is assigned prior to treatment 
and pathologic TNM is assigned based on clinical information plus information from surgery. The clinical TNM and the pathologic TNM values are 
summarized as clinical stage group or pathologic stage group.

For each cancer site, or schema, valid values, definitions, and registrar notes are provided for clinical TNM and stage group, pathologic TNM and stage 
group, and relevant Site-Specific Factors (SSFs).

TNM categories, stage groups, and definitions are based on the Union for International Cancer Control ([UICC](http://www.uicc.org/)) TNM 7th edition 
classification.  UICC 7th edition and AJCC 7th edition TNM categories and stage groups are very similar; however, there are some differences.

For diagnosis years 2016-2017, SEER Summary Stage 2000 is required. SEER Summary Stage 2000 should be collected manually unless the registry is collecting 
Collaborative Stage, which would derive Summary Stage 2000.

## Download

Java 8 is the minimum version required to use the library.

Download [the latest JAR][1] or grab via Maven:

```xml
<dependency>
    <groupId>com.imsweb</groupId>
    <artifactId>staging-algorithm-tnm</artifactId>
    <version>1.9.9</version>
</dependency>
```

or via Gradle:

```groovy
compile 'com.imsweb.com:staging-algorithm-tnm:1.9.9'
```

## Usage

Full detailed documentation can be found in the [Wiki](https://github.com/imsweb/staging-client-java/wiki/)

### Get a `Staging` instance

Everything starts with getting an instance of the `Staging` object.  There are `DataProvider` objects for each staging algorithm and version.  The `Staging`
object is thread safe and cached so subsequent calls to `Staging.getInstance()` will return the same object.

For example, to get an instance of the TNM algorithm

```java
Staging staging=Staging.getInstance(TnmDataProvider.getInstance(TnmVersion.V1_8));
```

### Schemas

Schemas represent sets of specific staging instructions.  Determining the schema to use for staging is based on primary site, histology and sometimes additional
discrimator values.  Schemas include the following information:

- schema identifier (i.e. "prostate")
- algorithm identifier (i.e. "tnm")
- algorithm version (i.e. "1.8")
- name
- title, subtitle, description and notes
- schema selection criteria
- input definitions describing the data needed for staging
- list of table identifiers involved in the schema
- a list of initial output values set at the start of staging
- a list of mappings which represent the logic used to calculate staging output

To get a list of all schema identifiers,

```java
Set<String> schemaIds = staging.getSchemaIds();
```

To get a single schema by identifer,

```java
Schema schema = staging.getSchema("prostate");
```

### Tables

Tables represent the building blocks of the staging instructions specified in schemas.  Tables are used to define schema selection criteria, input validation and staging logic.
Tables include the following information:

- table identifier (i.e. "ajcc7_stage")
- algorithm identifier (i.e. "tnm")
- algorithm version (i.e. "1.8")
- name
- title, subtitle, description, notes and footnotes
- list of column definitions
- list of table data

To get a list of all table identifiers,

```java
Set<String> tableIds = staging.getTableIds();
```

That list will be quite large.  To get a list of table indentifiers involved in a particular schema,

```java
Set<String> tableIds = staging.getInvolvedTables("prostate");
```

To get a single table by identifer,

```java
Table table = staging.getTable("ajcc7_stage");
```

### Lookup a schema

A common operation is to look up a schema based on primary site, histology and optionally one or more discriminators.  Each staging algorithm has a `SchemaLookup` object
customized for the specific inputs needed to lookup a schema.

For Collaborative Staging, use the `TnmSchemaLookup` object.  Here is a lookup based on site and histology.

```java
List<Schema> lookup = staging.lookupSchema(new TnmSchemaLookup("C629", "9231"));
assertEquals(1, lookup.size());
assertEquals("testis", lookup.get(0).getId());
```

If the call returns a single result, then it was successful.  If it returns more than one result, then it needs a discriminator.  Information about the required discriminator
is included in the list of results.  In the Collaborative Staging example, the field `ssf25` is always used as the discriminator.  Other staging algorithms may use different
sets of discriminators that can be determined based on the result.

```java
// do not supply a discriminator
List<StagingSchema> lookup = staging.lookupSchema(new TnmSchemaLookup("C111", "8200"));
assertEquals(2, lookup.size());
for (StagingSchema schema : lookup)
    assertTrue(schema.getSchemaDiscriminators().contains(TnmStagingData.SSF25_KEY));

// supply a discriminator
lookup = staging.lookupSchema(new TnmSchemaLookup("C111", "8200", "010"));
assertEquals(1, lookup.size());
assertEquals("nasopharynx", lookup.get(0).getId());
assertEquals(Integer.valueOf(34), lookup.get(0).getSchemaNum());
```

### Calculate stage

Staging a case requires first knowing which schema you are working with.  Once you have the schema, you can tell which fields (keys) need to be collected and supplied
to the `stage` method call.

A `StagingData` object is used to make staging calls.  All inputs to staging should be set on the `StagingData` object and the staging call will add the results.  The
results include:

- output - all output values resulting from the calculation
- errors - a list of errors and their descriptions
- path - an ordered list of the tables that were used in the calculation

Each algorithm has a specific `StagingData` entity which helps with preparing and evaluating staging calls.  For Collaborative Staging, use `TnmStagingData`.  One
difference between this library and the original Collaborative Stage library is that you do not have pass all 25 site-specific factors for every staging call. Only
include the ones that are used in the schema being staged.

```java
TnmStagingData data = new TnmStagingData();
data.setInput(TnmStagingData.TnmInput.PRIMARY_SITE, "C680");
data.setInput(TnmStagingData.TnmInput.HISTOLOGY, "8000");
data.setInput(TnmStagingData.TnmInput.BEHAVIOR, "3");
data.setInput(TnmStagingData.TnmInput.DX_YEAR, "2016");
data.setInput(TnmStagingData.TnmInput.RX_SUMM_SURGERY, "2");
data.setInput(TnmStagingData.TnmInput.RX_SUMM_RADIATION, "4");
data.setInput(TnmStagingData.TnmInput.REGIONAL_NODES_POSITIVE, "02");
data.setInput(TnmStagingData.TnmInput.CLIN_T, "c0");
data.setInput(TnmStagingData.TnmInput.CLIN_N, "c1");
data.setInput(TnmStagingData.TnmInput.CLIN_M, "c0");
data.setInput(TnmStagingData.TnmInput.PATH_T, "p0");
data.setInput(TnmStagingData.TnmInput.PATH_N, "p1");
data.setInput(TnmStagingData.TnmInput.PATH_M, "p1");

// perform the staging
staging.stage(data);

assertEquals(StagingData.Result.STAGED, data.getResult());
assertEquals("urethra", data.getSchemaId());
assertEquals(0, data.getErrors().size());
assertEquals(25, data.getPath().size());
assertEquals(10, data.getOutput().size());

// check outputs
assertEquals(TnmDataProvider.TnmVersion.LATEST.getVersion(), data.getOutput(TnmStagingData.TnmOutput.DERIVED_VERSION));
assertEquals("3", data.getOutput(TnmStagingData.TnmOutput.CLIN_STAGE_GROUP));
assertEquals("4", data.getOutput(TnmStagingData.TnmOutput.PATH_STAGE_GROUP));
assertEquals("4", data.getOutput(TnmStagingData.TnmOutput.COMBINED_STAGE_GROUP));
assertEquals("c0", data.getOutput(TnmStagingData.TnmOutput.COMBINED_T));
assertEquals("1", data.getOutput(TnmStagingData.TnmOutput.SOURCE_T));
assertEquals("c1", data.getOutput(TnmStagingData.TnmOutput.COMBINED_N));
assertEquals("1", data.getOutput(TnmStagingData.TnmOutput.SOURCE_N));
assertEquals("p1", data.getOutput(TnmStagingData.TnmOutput.COMBINED_M));
assertEquals("2", data.getOutput(TnmStagingData.TnmOutput.SOURCE_M));
```

## About SEER

The Surveillance, Epidemiology and End Results ([SEER](http://seer.cancer.gov)) Program is a premier source for cancer statistics in the United States. The SEER
Program collects information on incidence, prevalence and survival from specific geographic areas representing 28 percent of the US population and reports on all
these data plus cancer mortality data for the entire country.

[1]: http://repository.sonatype.org/service/local/artifact/maven/redirect?r=central-proxy&g=com.imsweb&a=staging-algorithm-tnm&v=LATEST
