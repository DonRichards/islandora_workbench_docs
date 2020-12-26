In addition to content files like images, the input data used by Workbench is a CSV file. This CSV file contains the metadata that is to be added to new or existing nodes, and some additional reserved columns specific to Workbench. Field values are either strings (for string or text fields), integers (for `field_weight`, for example), `1` or `0` for binary fields, Drupal-generated IDs (term IDs taxonomy terms or node IDs for collections and parents), or structured strings (for typed relation and geolocation fields).

!!! note
    As is standard with CSV data, field values do not need to be wrapped in double quotation marks (`"`), unless they contain an instance of the delimiter character (e.g., a comma). Spreadsheet applications such as Google Sheets and LibreOffice Calc will create valid CSV.
!!! note
    Also note that you can have Islandora Workbench [generate](/csv_file_templates/) a template CSV file for you.


The following types of fields can be used in your input CSV file:

* base fields
* text (plain, plain long, etc.) fields
* integer fields
* boolean fields, with values 1 or 0
* EDTF date fields
* entity reference (taxonomy and linked node) fields
* typed relation (taxonomy and linked node) fields
* geolocation fields

Where Drupal's configuration allows, fields in your input CSV can contain single values or multiple values, as described below.

### Required fields

* For the `create` task, `title`, `id` (or whatever field is identified in the `id_field` configuration option), and `file` are required. Empty values in the `file` field are allowed, in which case a node will be created but it will have no attached media.
* For the `update`, `delete`, and `add_media` tasks, the `node_id` field is required.
* For the `add_media` task, `file` is required, but for this task, `file` must contain a filename.

### Values in the "file" field

Values in the `file` field in your CSV can be relative to the directory named in `input_dir`, abolute paths, or URLs. Examples of each:

* relative to directory named in `input_dir`: `myfile.png`
* absolute: `/tmp/data/myfile.png`
* URL: `http://example.com/files/myfile.png`

Relative, absolute, and URL file locations can exist within the same CSV file.

Things to note about URL paths:

* Workbench downloads files identified by URLs and saves them in the directory named in `input_dir` before processing them further. It does not delete the files after they have been ingested into Islandora.
* Files identified by URLs must be accessible to the Workbench script, which means they must not require a username/password; however, they can be protected by a firewall, ACL, etc. as long as the computer running Workbench is allowed to retrieve the files without authenticating.
* Currently Workbench requires that the URLs point directly to a file and not to a script, wrapper page, or other indirect route to the file.

### Base fields

Base fields are basic node properties, shared by all content types. The base fields you can include in your CSV file are:

* `title`: This field is required for all rows in your CSV for the `create` task. Optional for the 'update' task. Drupal limits the title's length to 255 characters, and Workbench will check that titles are less than 255 characters unless your configuration file contains `validate_title_length: False` as described above.
* `langcode`: The language of the node. Optional. If included, use one of Drupal's language codes as values (common values are 'en', 'fr', and 'es'; the entire list can be seen [here](https://git.drupalcode.org/project/drupal/-/blob/8.8.x/core/lib/Drupal/Core/Language/LanguageManager.php#L224). If absent, Drupal sets the value to the default value for your content type.
* `uid`: The Drupal user ID to assign to the node and media created with the node. Optional. Only available in `create` tasks. If you are creating paged/compound objects from directories, this value is applied to the parent's children (if you are creating them using the page/child-level metadata method, these fields must be in your CSV metadata).
* `created`: The timestamp to use in the node's "created" attribute and in the "created" attribute of the media created with the node. Optional, but if present, it must be in format 2020-11-15T23:49:22+00:00 (the +00:00 is the difference to Greenwich time/GMT). Only available in `create` tasks. If you are creating paged/compound objects from directories, this value is applied to the parent's children (if you are creating them using the page/child-level metadata method, these fields must be in your CSV metadata).

### Single-valued fields

You can include additional fields that will be added to the nodes. The column headings in the CSV file must match machine names of fields that exist in the target Islandora content type.

For example, using the fields defined by the Islandora Defaults module for the "Repository Item" content type, your CSV file could look like this:

```csv
file,title,id,field_model,field_description,field_rights,field_extent,field_access_terms,field_member_of
myfile.jpg,My nice image,obj_00001,24,"A fine image, yes?",Do whatever you want with it.,There's only one image.,27,45
```

In this example, the term ID for the tag you want to assign in `field_access_terms` is 27, and the node ID of the collection you want to add the object to (in `field_member_of`) is 45.

### Multivalued fields

For multivalued fields, separate the values within a field with a pipe (`|`), like this:

```
file,title,field_my_multivalued_field
IMG_1410.tif,Small boats in Havana Harbour,foo|bar
IMG_2549.jp2,Manhatten Island,bif|bop|burp
```

This works for string fields as well as reference fields, e.g.:

```
file,title,field_my_multivalued_taxonomy_field
IMG_1410.tif,Small boats in Havana Harbour,35|46
IMG_2549.jp2,Manhatten Island,34|56|28
```

Drupal strictly enforces the maximum number of values allowed in a field. If the number of values in your CSV file for a field exceed a field's configured maximum number of fields, Workbench will only populate the field to the field's configured limit.

The subdelimiter character defaults to a pipe (`|`) but can be set in your config file using the `subdelimiter: ";"` option.


### Taxonomy fields

In CSV columns for taxonomy fields, you can use either term IDs (integers) or term names (strings). You can even mix IDs and names in the same field:

```
file,title,field_my_multivalued_taxonomy_field
img001.png,Picture of cats and yarn,Cats|46
img002.png,Picture of dogs and sticks,Dogs|Sticks
img003.png,Picture of yarn and needles,"Yarn, Balls of"|Knitting needles
```
By default, if you use a term name in your CSV data that doesn't match a term name that exists in the referenced taxonomy, Workbench will detect this when you use `--check` and exit. However, if you add `allow_adding_terms: true` to your configuration file for `create` and `update` tasks, Workbench will create the new term. A few of things to note:

* To create new terms, your target Drupal needs to have its "Taxonomy term" REST endpoint enabled as described in the "Requirements" section at the beginning of this README.
* If multiple records in your CSV contain the same new term name in the same field, the term is only created once.
* When Workbench checks to see if the term with the new name exists in the target vocabulary, it normalizes it and compares it with existing term names in that vocabulary, applying these normalization rules to both the new term and the existing terms:
    * It strips all leading and trailing whitespace.
    * It replaces all other whitespace with a single space character.
    * It converts all text to lower case.
    * It removes all punctuation.
    * If the term name you provide in the CSV file does not match any existing term names in the vocabulary linked to the field after these normalization rules are applied, it is used to create a new taxonomy term. If it does match, Workbench populates the field in your nodes with the matching term.

Adding new terms has some contraints:

* Creating taxonomy terms by including them in your CSV file adds new terms to the root of the applicable vocabulary. Workbench cannot create a new term that has another term as its parent (i.e. terms below the top leve of a hierarchical taxonomy). However, for existing terms, Workbench doesn't care where they are in a taxonomy's hierarchy.
* Terms created in this way do not have any external URIs. If you want your terms to have external URIs, you will need to either create the terms manually or add the URIs manually after the terms are created by Islandora Workbench.
* `--check` will identify any new terms that exceed Drupal's maxiumum allowed length for term names, 255 characters. If a term name is longer than 255 characters, Workbench will truncate it at that length, log that it has done so, and create the term.
* Taxonomy terms created with new nodes are not removed when you delete the nodes.

#### Using term names in multi-vocabulary fields

While most node taxonomy fields reference only a single vocabulary, Drupal does allow fields to reference multiple vocabularies. This ability poses a problem when we use term names instead of term IDs in our CSV files: in a multi-vocabulary field, Workbench can't be sure which term name belongs in which of the multiple vocabularies referenced by that field. This applies to both existing terms and to new terms we want to add when creating node content.

To avoid this problem, we need to tell Workbench which of the multple vocabularies each term name should (or does) belong to. We do this by namespacing terms with the applicable vocabulary ID.

For example, let's imagine we have a node field whose name is `field_sample_tags`, and this field references two taxonomies, `cats` and `dogs`. To use the terms `Tuxedo`, `Tabby`, `German Shepherd` in the CSV when adding new nodes, we would namespace them with vocabulary IDs like this:


```
field_sample_tags
cats:Tabby
cats:Tuxedo
dogs:German Shepherd
```

If you want to use multiple terms in a single field, you would namespace them all:

```
cats:Tuxedo|cats:Misbehaving|dogs:German Shepherd
```

Term names containing commas (`,`) in multi-valued, multi-vocabulary fields need special treatment (no surprise there): you need to wrap the entire field in quotation marks (like you would for any other CSV value that contains a comma), and in addition, specify the namespace within each of the values:

```
"tags:gum, Bubble|tags:candy, Hard"
```
Using these conventions, Workbench will be certain which vocabulary the term names belong to. Workbench will remind you during its `--check` operation that you need to namespace terms. It determines 1) if the field references multiple vocabularies, and then checks to see 2) if the field's values in the CSV are term IDs or term names. If you use term names in multi-vocabulary fields, and the term names aren't namespaced, Workbench will warn you:

```
Error: Term names in multi-vocabulary CSV field "field_tags" require a vocabulary namespace; value "Dogs" in row 4 does not have one.
```

Note that since `:` is a special character when you use term names in multi-vocabulary CSV fields, you can't add a namespaced term that itself contains a `:`. You need to add it manually to Drupal and then use its term ID (or name, or URI) in your CSV file.

#### Using term URIs instead of term IDs

Islandora Workbench lets you use URIs assigned to terms instead of term IDs. You can use a term URI in the `media_use_tid` configuration option (for example, `"http://pcdm.org/use#OriginalFile"`) and in taxonomy fields in your metadata CSV file:

```
field_model
https://schema.org/DigitalDocument
http://purl.org/coar/resource_type/c_18cc
```

During `--check`, Workbench will validate that URIs correspond to existing taxonomy terms.

Using term URIs has some constraints:

* You cannot create a new term by providing a URI like you can by providing a term name.
* If the same URI is registered with more than one term, Workbench will choose one and write a warning to the log indicating which term it chose and which terms the URI is registered with. However, `--check` will detect that a URI is registered with more than one term and warn you. At that point you can edit your CSV file to use the correct term ID rather than the URI.

### Typed Relation fields

Unlike most field types, which take a string or an integer as their value in the CSV file, fields that have the "Typed Relation" type take structured values that need to be entered in a specific way in the CSV file. An example of this type of field is the "Linked Agent" field in the Repository Item content type created by the Islandora Defaults module.

!!! note
    In the following section, the word "namespace" is used to mean something slightly different than what it means in the above section. Below, "namespace" is used both to refer to a vocabulary ID in the target Drupal and an ID for the authority list of relators maintained outside of Drupal.

The structure of values for this field encode a namespace (indicating the external authority list the relation is from), a relation type, and a target (which identifies what the relation refers to, such as a specific taxonomy term in the target Drupal), each separated by a colon (`:`). The first two parts, the namespace and the relation type, come from the "Available Relations" section of the field's configuration, which looks like this (using the "Linked Agent" field's configuration as an example):

![Relations example](images/relators.png)

In the node edit form, this structure is represented as a select list of the types (the namespace is not shown) and, below that, an autocomplete field to indicate the relation target, e.g.:

![Linked agent example](images/linked_agent.png)

To include these kind of values in a CSV field, we need to use a structured string as described above (namespace:relationtype:targetid). For example:

`relators:art:30`

You can also use taxonomy term names as targets:

`"relators:art:Jordan, Mark"`

If the field you are populating references multiple vocabularies, you must include a vocabulary identifer with your term name:

`"relators:art:person:Jordan, Mark"`

You can also use HTTP URIs as typed relation targets:

`"relators:art:http://markjordan.net`

> Note that the structure required for typed relation values in the CSV file is not the same as the structure of the relations configuration depicted in the first screenshot above; the CSV values use only colons to seprate the three parts, but the field configuration within Drupal uses a colon and then a pipe (|) to structure its values.

In this example of a CSV value, `relators` is the namespace that the relation type `art` is from (the Library of Congress [Relators](http://id.loc.gov/vocabulary/relators.html) vocabulary), and the target taxonomy term ID is `30`. In the screenshot above showing the "Linked Agent" field of a node, the value of the Relationship Type select list is "Artist (art)", and the value of the associated taxonomy term field is the person's name that has the taxonomy term ID "30" (in this case, "Jordan, Mark"):

If you want to include multiple typed relation values in a single field of your CSV file (such as in "field_linked_agent"), separate the three-part values with the same subdelimiter character you use in other fields, e.g. (`|`) (or whatever you have configured as your `subdelimiter`):

`relators:art:30|relators:art:45`

#### Adding new typed relation targets

Islandora Workbench allows you to add new typed relation targets while creating and updating nodes. These targets are taxonomy terms. Your configuration file must include the `allow_adding_terms: true` option to add new targets. In general, adding new typed relation targets is just like adding new taxonomy terms as described above in the "Taxonomy fields" section.

An example of a CSV value that adds a new target term is:

`"relators:art:person:Jordan, Mark"`

You can also add multiple new targets:

`"relators:art:person:Annez, Melissa|relators:art:person:Jordan, Mark"`

Note that:

* For multi-vocabulary fields, new typed relator targets must be accommpanied by a vocabulary namespace (`person` in the above examples).
* You cannot add new relators (e.g. `relators:foo`) in your CSV file, only new target terms.

### Geolocation fields

The Geolocation field type, managed by the [Geolocation Field](https://www.drupal.org/project/geolocation) contrib module, stores latitude and longitude coordinates in separate data elements. To add or update fields of this type, Workbench needs to provide the latitude and longitude data in these separate elements.

To simplify entering geocoordinates in the CSV file, Workbench allows geocoordinates to be in `lat,lng` format, i.e., the latitude coordinate followed by a comma followed by the longitude coordinate. When Workbench reads your CSV file, it will split data on the comma into the required lat and lng parts. An example of a single geocoordinate in a field would be:

```
field_coordinates
"49.16667,-123.93333"
```

You can include multiple pairs of geocoordinates in one CSV field if you separate them with the subdelimiter character:

```
field_coordinates
"49.16667,-123.93333|49.25,-124.8"
```

Note that:

* Geocoordinate values in your CSV need to be wrapped in double quotation marks, unless the `delimiter` key in your configuration file is set to something other than a comma.
* If you are entering geocoordinates into a spreadsheet, a leading `+` will make the spreadsheet application think you are entering a formula. You can work around this by escaping the `+` with a backslash (`\`), e.g., `49.16667,-123.93333` should be `\+49.16667,-123.93333`, and `49.16667,-123.93333|49.25,-124.8` should be `\+49.16667,-123.93333|\+49.25,-124.8`. Workbench will strip the leading `\` before it populates the Drupal fields.

