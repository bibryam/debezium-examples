# A Simple Java Client

## Prerequisites: run postgresql instructions

Runs a simple query against a JDBC or ODBC endpoints using a Spring JDBC template.  

Edit the src/main/resources/application.properties and/or one of the profile properties to match the username, password, and url of the PD instance you are testing against.


Build and run against ODBC (35432) endpoint

```
mvn -Dspring.profiles.active=postgresql
```

