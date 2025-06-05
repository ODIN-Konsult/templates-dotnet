# syntax=docker/dockerfile:1.7-labs
ARG DOTNET_VERSION=latest
ARG CONFIGURATION=Release
ARG SOLUTION_NAME="*"
ARG SOLUTION_EXTENSION="slnx"
ARG PUBLISHED_PROJECT=""
ARG DATABASE_PROJECT=""
ARG DBCONTEXT=""

FROM mcr.microsoft.com/dotnet/runtime:${DOTNET_VERSION} AS runtime
WORKDIR /app

FROM mcr.microsoft.com/dotnet/aspnet:${DOTNET_VERSION} AS asp
WORKDIR /app

FROM mcr.microsoft.com/dotnet/sdk:${DOTNET_VERSION} AS sdk
ARG CONFIGURATION
ENV DOTNET_NOLOGO=0
ENV DOTNET_CLI_TELEMETRY_OPTOUT=1
WORKDIR /build
COPY .git/ ./.git/

FROM sdk AS restore
ARG SOLUTION_NAME
ARG SOLUTION_EXTENSION

COPY --link ${SOLUTION_NAME}.${SOLUTION_EXTENSION} Directory.Build.* Directory.Packages.props ./
COPY --link --parents ./**.csproj ./**/packages.lock.json ./

RUN dotnet restore

FROM restore AS build
COPY src src
COPY tests tests
COPY samples samples

RUN dotnet build --no-restore --configuration ${CONFIGURATION}

FROM build AS test
RUN dotnet test --no-build --configuration ${CONFIGURATION}

FROM test AS publish
ARG PUBLISHED_PROJECT
ENV PROJECT=src/${PUBLISHED_PROJECT}/${PUBLISHED_PROJECT}.csproj
RUN dotnet publish --no-build --configuration ${CONFIGURATION} -o /app/publish ${PROJECT}

FROM build AS pack
RUN dotnet pack --no-build --configuration Release --output /app/publish

FROM build AS migration-bundle
ARG PUBLISHED_PROJECT
ARG DATABASE_PROJECT
ARG DBCONTEXT
ENV CONTEXT="${DBCONTEXT:+--context ${DBCONTEXT}}"
ENV EFPROJECT=src/${DATABASE_PROJECT}/${DATABASE_PROJECT}.csproj
ENV RUNNER=src/${PUBLISHED_PROJECT}/${PUBLISHED_PROJECT}.csproj
ENV PATH="${PATH}:/root/.dotnet/tools"

RUN dotnet tool install --global dotnet-ef
RUN dotnet ef migrations bundle --configuration ${CONFIGURATION} --no-build --project ${EFPROJECT} --startup-project ${RUNNER} --output /app/migrations ${CONTEXT}

FROM asp as migrations
ENV MIGRATION_TARGET=""
COPY --link --chown=root:root --chmod=755 --from=migration-bundle /app/migrations .
USER app
ENTRYPOINT ./migrations ${MIGRATION_TARGET}

FROM runtime AS nuget
COPY --link --chown=root:root --chmod=644 --from=pack /app/publish .

FROM runtime as final-app
ARG PUBLISHED_PROJECT
ENV PROJECT=${PUBLISHED_PROJECT}
COPY --link --chown=root:root --from=publish /app/publish .
USER app
ENTRYPOINT dotnet ${PROJECT}.dll

FROM asp as final-web
ARG PUBLISHED_PROJECT
ENV PROJECT=${PUBLISHED_PROJECT}
COPY --link --chown=root:root --from=publish /app/publish/ .
USER app
ENTRYPOINT dotnet ${PROJECT}.dll
