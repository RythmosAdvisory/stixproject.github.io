require 'bundler'
require 'fileutils'
require 'liquid'
require 'stix_schema_spy'

$destination = "data-model"

$practices = {
    "indicator:IndicatorType" => "sp_indicator.md",
    "marking:MarkingSpecificationType" => "sp_handling.md",
    "stix:STIXHeaderType" => "sp_package.md",
    "stix:STIXType" => "sp_package.md",
}

desc "Regenerate the data model documentation"
task :regenerate do
  # Make Liquid aware of the Jekyll includes directory
  #Liquid::Template.file_system = Liquid::LocalFileSystem.new("_includes")

  # Preload all versions of all schemas first so our introspection can tell what's available
  StixSchemaSpy::Schema::VERSIONS.each {|v| StixSchemaSpy::Schema.preload!(v)}

  StixSchemaSpy::Schema::VERSIONS.each do |version|
    # Load the documentation page template
    template = File.read("_layouts/data_model_page.html")

    # Write the file for the data model autocompleter
    json = StixSchemaSpy::Schema.all(version).map {|schema| schema.complex_types}.flatten.map {|type| {:name => type.name, :schema => type.schema.title, :link => type_link(type, version)}}
    File.open("js/autocomplete-#{version}.js", "w") {|f| f.write("window.typeSuggestions = " + JSON.dump(json))}

    # Iterate through all types in the schemas and create pages for them
    StixSchemaSpy::Schema.all(version).each do |schema|
      schema.complex_types.each do |type|
        write_page(type, template, version)
      end
    end
  end
end

def write_page(type, template, version)
  destination = type_path(type, version)

  FileUtils.mkdir_p(destination)

  results = Liquid::Template.parse(template).render(
    'type' => {
      'latest_version' => StixSchemaSpy::Schema.latest_version,
      'this_version' => version,
      'versions' => StixSchemaSpy::Schema::VERSIONS.reject {|v|
        schema = StixSchemaSpy::Schema.find(type.schema.prefix, v)
        (schema && schema.find_type(type.name)).nil?
      },
      'name' => type.name,
      'documentation' => process_documentation(type.documentation, version),
      'schema' => {
        'title' => type.schema.title,
        'prefix' => type.schema.prefix
      },
      'vocab?' => type.vocab?,
      'fields?' => type.fields.length > 0,
      'fields' => fields(type, version),
      'vocab_items' => vocab_items(type),
      'suggested_practices' => suggested_practices(type)
    }
  )

  File.open("#{destination}/index.html", "w") {|f| f.write(results)}
end

def type_link(type, version)
  "/#{type_path(type, version)}"
end

def type_path(type, version)
  "#{$destination}/#{version}/#{type.schema.prefix}/#{type.name}"
end

def suggested_practices(type)
  $practices["#{type.schema.prefix}:#{type.name}"]
end

def fields(type, version)
  type.fields.map do |field|
    {
      'name' => field.name,
      'link' => (field.type.kind_of?(StixSchemaSpy::ComplexType) && field.type.schema && !field.type.schema.blacklisted?) ? type_link(field.type, version) : false,
      'type' => field.type.name,
      'documentation' => process_documentation(field.documentation.split("\n"), version),
      'occurrence' => field_occurrence(field)
    }
  end
end

def field_occurrence(field)
  if field.kind_of?(StixSchemaSpy::Attribute)
    field.use
  else
    max_occurs = field.max_occurs == 'unbounded' ? 'n' : field.max_occurs
    "#{field.min_occurs}..#{max_occurs}"
  end
end

def vocab_items(type)
  return [] unless type.vocab?

  type.vocab_values.map {|v|
    {
      'name' => v[0],
      'description' => v[1]
    }
  }
end

def process_documentation(docs, version)
  if docs.kind_of?(String)
    add_internal_links(docs, version)
  else
    docs.map {|doc|
      add_internal_links(doc, version)
    }
  end
end

def add_internal_links(doc, version)
  doc
    .gsub(/\S+Vocab-\d(\.\d){1,2}/) {|match| "<a href='/#{$destination}/#{version}/stixVocabs/#{match}'>#{match}</a>"}
    .gsub(/ \S+Type /) do |match|
      name = match.strip
      type = find_type(name)
      if type
        " <a href='#{type_link(type, version)}'>#{name}</a> "
      else
        match
      end
    end
end

def find_type(type)
  types = StixSchemaSpy::Schema.all.map do |schema|
    schema.find_type(type)
  end.compact

  # Only return the found type if we found exactly one, otherwise it's ambiguous
  types.length == 1 ? types.first : nil
end
