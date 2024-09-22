# Fixtures

A *fixture* is a collection of files that contain the serialized contents of the database.  

Fixtures can be written as:

* JSON
* XML
* YAML (with PyYAML installed)


Django will search in these locations for fixtures:

* In the fixtures directory of every installed application
* In any directory listed in the `FIXTURE_DIRS` setting
* In the literal path named by the fixture
