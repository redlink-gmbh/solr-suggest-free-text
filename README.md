# Redlink Free Text Suggester

[![Build Status](https://travis-ci.org/redlink-gmbh/solr-suggest-free-text.svg?branch=master)](https://travis-ci.org/redlink-gmbh/solr-suggest-free-text)
[![Maven Central](https://img.shields.io/maven-central/v/io.redlink.solr/solr-suggest-free-text.svg)](http://search.maven.org/#search%7Cga%7C1%7Cg%3A%22io.redlink.solr%22)
[![Sonatype (Snapshots)](https://img.shields.io/nexus/s/https/oss.sonatype.org/io.redlink.solr/solr-suggest-free-text.svg)](https://oss.sonatype.org/#nexus-search;gav~io.redlink.solr~solr-suggest-free-text~~~)

## Overview:

A Suggester based on the [Solr FreeTextLookupFactory](https://lucene.apache.org/solr/guide/7_7/suggester.html#freetextlookupfactory) that returns the collates the parsed query with suggestions obtained by the same way as implemented by the Free Text Suggester. To improve the user expirience the suggested part of the returned query is highlighted.

## Supported Solr Versions

This module depends on APIs that have changed in an incompatible manner between Solr 7 and Solr 8.

* `1.x` releases are compatible to Solr `7.x` releases
* `2.x` releases are compatible to Solr `8.x` releases (currently tested up to Solr `8.4.1`)

## Configuration

This implementation supports the exact same configuration parameter as the original Solr [FreeTextLookupFactory](https://lucene.apache.org/solr/guide/7_7/suggester.html#freetextlookupfactory).

Important:

* set the `lookupImpl` to `io.redlink.lucene.search.suggest.FreeTextSuggesterFactory` to use this suggester
* set the `suggestFreeTextAnalyzerFieldType` to a suitable field type. Remember this suggester suggest terms, so returned values will not be the original tokens, but the analyzed terms! So minor normalizations like lower case will be ok, but other things like stemming, ASKII folding .. will most likely not result in suggestions expected by users
* set the `field` parameter to the field providing the content for the suggester. This implementation does work with content (e.g. articles title, teaser and content) so creating a field with copy field configurations in the `schema.xml` might be the way to go. The field type for this field does not matter. It just needs to be `stored="true"`.
* set `buildOnStartup` and `buildOnCommit` to `false` as building the data structure for this suggester is expensive. Use explicit requests to a `requestHandler` using this `searchComponent` with the parameter `suggest.build=true` to explicitly trigger the building of the index.
    * also think about blocking `suggest.build`, `suggest.buildAll` on publicly exposed `requestHandler` as multiple request with `suggest.build=true` could be used for DOS attacks on your search service. To do this just set those parameters to `false` in the `invariants` of that endpoint. To build the suggester index configure an additional endpoint that is not publicly available.

Example configuration:

```
<searchComponent name="suggest" class="solr.SuggestComponent">
    <lst name="suggester">
        <str name="name">contentSuggester</str>
        <str name="lookupImpl">io.redlink.lucene.search.suggest.FreeTextSuggesterFactory</str>
        <str name="dictionaryImpl">DocumentDictionaryFactory</str>
        <str name="field">content_de</str>
        <str name="suggestFreeTextAnalyzerFieldType">text_auto_complete_de</str>
        <str name="ngrams">3</str>
        <str name="storeDir">contentSuggesterData</str>
        <str name="buildOnStartup">false</str>
        <str name="buildOnCommit">false</str>
    </lst>
</searchComponent>
```

and the public suggest endpoint

```
<requestHandler name="/suggest" class="solr.SearchHandler">
  <lst name="defaults">
    <str name="suggest">true</str>
    <str name="suggest.count">5</str>
  </lst>
  <lst name="invariants">
    <str name="wt">json</str>
    <str name="json.nl">map</str>
    <!-- disable building/releod the suggest index via the public endpoint -->
    <str name="suggest.build">false</str>
    <str name="suggest.reload">false</str>
    <str name="suggest.buildAll">false</str>
    <str name="suggest.reloadAll">false</str>
  </lst>  <arr name="components">
    <str>suggest</str>
  </arr>
</requestHandler>

```

finally a second (private) requestHandler just used to build the suggest index


```
<requestHandler name="/rebuild/suggest" class="solr.SearchHandler">
    <lst name="invariants">
        <str name="wt">json</str>
        <str name="json.nl">map</str>
        <str name="suggest">true</str>
        <str name="suggest.buildAll">true</str>
        <str name="rows">0</str>
    </lst>
    <arr name="last-components">
        <str>suggest</str>
    </arr>
</requestHandler>
```


## Response Format

It uses the default suggester response format of Solr:

* The `term` field contains the suggested value including highlighting
* The `payload` field the text to be used in a query when the user accepts a suggestion

Example response for the request `Ganztag kindergar`

```
"suggest":{"contentSuggester":{
      "Ganztag kindergar":{
        "numFound":10,
        "suggestions":[{
            "term":"ganztag\u001e<em>kindergarten</em>",
            "weight":1125667294798616,
            "payload":"ganztag\u001ekindergarten"},
          {
            "term":"ganztag\u001e<em>kindergartens</em>",
            "weight":66967949675612,
            "payload":"ganztag\u001ekindergartens"},
          [..],
          {
            "term":"ganztag\u001e<em>kindergartenausstattung</em>",
            "weight":5720880123042,
            "payload":"ganztag\u001ekindergartenausstattung"}]}}}

```

 ## License
 
 [Apache License, Version 2.0](https://www.apache.org/licenses/LICENSE-2.0)
 
