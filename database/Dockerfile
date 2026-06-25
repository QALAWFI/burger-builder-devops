FROM postgres:16-alpine

ENV POSTGRES_DB=burgerbuilder
ENV POSTGRES_USER=postgres
ENV POSTGRES_PASSWORD=YourStrong!Passw0rd

# Scripts in /docker-entrypoint-initdb.d run once, in alphabetical order,
# only when the data directory is empty (first container start)
COPY schema-postgresql.sql /docker-entrypoint-initdb.d/01-schema.sql
COPY data-postgresql.sql /docker-entrypoint-initdb.d/02-data.sql

EXPOSE 5432
