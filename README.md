# pgclimb [![Build Status](https://travis-ci.org/lukasmartinelli/pgclimb.svg?branch=master)](https://travis-ci.org/lukasmartinelli/pgclimb) [![Go Report Card](https://goreportcard.com/badge/github.com/lukasmartinelli/pgclimb)](https://goreportcard.com/report/github.com/lukasmartinelli/pgclimb) ![License](https://img.shields.io/badge/license-MIT-blue.svg)

<img align="right" alt="Climbing elephant" src="logo.png" />

A PostgreSQL utility to export data into different data formats with
support for templates.

Features:
- Export data to [JSON](#json-document), [JSON Lines](#json-lines), [CSV](#csv-and-tsv), [XLSX](#xlsx), [XML](#xml)
- Use [Templates](#templates) to support custom formats (HTML, Markdown, Text)

Use Cases:
- `psql` alternative for getting data out of PostgreSQL
- Publish data sets
- Create Excel reports from the database
- Generate HTML reports
- Export XML data for further processing with XSLT
- Transform data to JSON for graphing it with JavaScript libraries
- Generate readonly JSON APIs

## Install (mokalovesoulmate custom)

- `go mod vendor`
- `go build` or `gox` for cross-platform compile

## Install (Original notes)

You can download a single binary for Linux, OSX or Windows.

**OSX**

```bash
wget -O pgclimb https://github.com/lukasmartinelli/pgclimb/releases/download/v0.3/pgclimb_darwin_amd64
chmod +x pgclimb

./pgclimb --help
```

**Linux**

```bash
wget -O pgclimb https://github.com/lukasmartinelli/pgclimb/releases/download/v0.3/pgclimb_linux_amd64
chmod +x pgclimb

./pgclimb --help
```

**Install from source**

```bash
go get github.com/lukasmartinelli/pgclimb
```

If you are using Windows or 32-bit architectures you need to [download the appropriate binary
yourself](https://github.com/lukasmartinelli/pgclimb/releases/latest).

## Supported Formats

The example queries operate on the open data [employee salaries of Montgomery County Maryland](https://data.montgomerycountymd.gov/api/views/54rh-89p8/rows.csv). You can import CSV files into your database using [my PostgreSQL import tool pgfutter](http://github.com/lukasmartinelli/pgfutter).
To connect to your beloved PostgreSQL database set the [appropriate connection options](#database-connection).

### CSV and TSV

Exporting CSV and TSV files is very similar to using `psql` and the `COPY TO` statement.

```bash
# Write CSV file to stdout with comma as default delimiter
pgclimb -c "SELECT * FROM employee_salaries" csv

# Save CSV file with custom delimiter and header row to file
pgclimb -o salaries.csv \
    -c "SELECT full_name, position_title FROM employee_salaries" \
    csv --delimiter ";" --header

# Create TSV file with SQL query from stdin
pgclimb -o positions.tsv tsv <<EOF
SELECT position_title, COUNT(*) FROM employee_salaries
GROUP BY position_title
ORDER BY 1
EOF
```

### JSON Document

Creating a single JSON document of a query is helpful if you
interface with other programs like providing data for JavaScript or creating
a readonly JSON API. You don't need to `json_agg` your objects, `pgclimb` will
automatically serialize the JSON for you - it also supports nested JSON objects for more complicated queries.

```bash
# Query all salaries into JSON array
pgclimb -c "SELECT * FROM employee_salaries" json

# Query all employees of a position as nested JSON object
cat << EOF > employees_by_position.sql
SELECT s.position_title, json_agg(s) AS employees
FROM employee_salaries s
GROUP BY s.position_title
ORDER BY 1
EOF

# Load query from file and store it as JSON array in file
pgclimb -f employees_by_position.sql \
    -o employees_by_position.json \
    json
```

### JSON Lines

[Newline delimited JSON](http://jsonlines.org/) is a good format to exchange
structured data in large quantities which does not fit well into the CSV format.
Instead of storing the entire JSON array each line is a valid JSON object.

```bash
# Query all salaries as separate JSON objects
pgclimb -c "SELECT * FROM employee_salaries" jsonlines

# In this example we interface with jq to pluck the first employee of each position
pgclimb -f employees_by_position.sql jsonlines | jq '.employees[0].full_name'
```

### XLSX

Excel files are really useful to exchange data with non programmers
and create graphs and filters. You can fill different datasets into different spreedsheets and distribute one single Excel file.

```bash
# Store all salaries in XLSX file
pgclimb -o salaries.xlsx -c "SELECT * FROM employee_salaries" xlsx

# Create XLSX file with multiple sheets
pgclimb -o salary_report.xlsx \
    -c "SELECT DISTINCT position_title FROM employee_salaries" \
    xlsx --sheet "positions"
pgclimb -o salary_report.xlsx \
    -c "SELECT full_name FROM employee_salaries" \
    xlsx --sheet "employees"
```

### XML

You can output XML to process it with other programs like [XSLT](http://www.w3schools.com/xml/xsl_intro.asp).
To have more control over the XML output you should use the `pgclimb` template functionality directly to generate XML or build your own XML document with [XML functions in PostgreSQL](https://wiki.postgresql.org/wiki/XML_Support).

```bash
# Output XML for each row
pgclimb -o salaries.xml -c "SELECT * FROM employee_salaries" xml
```

A good default XML export is currently lacking because the XML format
can be controlled using templates.
If there is enough demand I will implement a solid
default XML support without relying on templates.

## Templates

Templates are the most powerful feature of `pgclimb` and allow you to implement
other formats that are not built in. In this example we will create a
HTML report of the salaries.

Create a template `salaries.tpl`.

```html
<!DOCTYPE html>
<html>
    <head><title>Montgomery County MD Employees</title></head>
    <body>
        <h2>Employees</h2>
        <ul>
            {{range .}}
            <li>{{.full_name}}</li>
            {{end}}
        </ul>
    </body>
</html>
```

And now run the template.

```
pgclimb -o salaries.html \
    -c "SELECT * FROM employee_salaries" \
    template salaries.tpl
```

## Database Connection

Database connection details can be provided via environment variables
or as separate flags (same flags as `psql`).

name        | default     | flags               | description
------------|-------------|---------------------|-----------------
`DB_NAME`   | `postgres`  | `-d`, `--dbname`    | database name
`DB_HOST`   | `localhost` | `--host`            | host name
`DB_PORT`   | `5432`      | `-p`, `--port`      | port
`DB_USER`   | `postgres`  | `-U`, `--username`  | database user
`DB_PASS`   |             | `--pass`            | password (or empty if none)

## Advanced Use Cases

### Different ways of Querying

Like `psql` you can specify a query at different places.

```bash
# Read query from stdin
echo "SELECT * FROM employee_salaries" | pgclimb
# Specify simple queries directly as arguments
pgclimb -c "SELECT * FROM employee_salaries"
# Load query from file
pgclimb -f query.sql
```

### Control Output

`pgclimb` will write the result to `stdout` by default.
By specifying the `-o` option you can write the output to a file.

```bash
pgclimb -o salaries.tsv -c "SELECT * FROM employee_salaries" tsv
```

### Using JSON aggregation

This is not a `pgclimb` feature but shows you how to create more complex
JSON objects by using the [PostgreSQL JSON functions](http://www.postgresql.org/docs/9.5/static/functions-json.html).

Let's query communities and join an additional birth rate table.

```bash
pgclimb -c "SELECT id, name, \\
    (SELECT array_to_json(array_agg(t)) FROM ( \\
            SELECT year, births FROM public.births \\
            WHERE community_id = c.id \\
            ORDER BY year ASC \\
        ) AS t \\
    ) AS births, \\
    FROM communities) AS c" jsonlines
```

# Contribute

## Dependencies

Go get the required dependencies for building `pgclimb`.

```bash
go get github.com/codegangsta/cli
go get github.com/lib/pq
go get github.com/jmoiron/sqlx
go get github.com/tealeg/xlsx
go get github.com/andrew-d/go-termutil
```

## Cross-compiling

We use [gox](https://github.com/mitchellh/gox) to create distributable
binaries for Windows, OSX and Linux.

```bash
docker run --rm -v "$(pwd)":/usr/src/pgclimb -w /usr/src/pgclimb tcnksm/gox:1.9
```

## Integration Tests

Run `test.sh` to run integration tests of the program with a PostgreSQL server. Take a look at the `.travis.yml`.