# Map Service Search
This little web application enables you to search for datasources within MapServices of various ArcGIS Servers.

The app can be hosted on-premise and there is a small demo available ([take me there](https://mu88.github.io/MapServiceSearch/index.html)).

## Used technologies/components
The app is made up of three different components:
* Client
  * Web application based on [AngularJS](https://angularjs.org/) and [Bootstrap](http://getbootstrap.com/), embedded into an ASP.NET Web Application
* Backend
  * Search index based on [Elasticsearch](https://www.elastic.co) that can be accessed via its REST API
* FME (if you don't know FME: it has a super simple approach and you can apply for a [Free Home Use License](https://www.safe.com/free-fme-licenses/home-use/) :thumbsup:)
  * Workspace is based on [FME Desktop 2018](https://www.safe.com/fme/fme-desktop/)
  * Extraction of the datasources from MapServices hosted on several ArcGIS Servers

## How it works
In the FME Workspace, all ArcGIS Servers to analyze are stored as a JSON object, so the list of ArcGIS Servers to process can be extended easily. For each ArcGIS Server the Workspace retrieves all MapServices. Afterwards, all the relevant information of each MapService are extracted and combined. By doing this, the processing provides the following information:
* Name of the MapService
* Environment of the ArcGIS Server (e. g. *Staging* or *Production*)
* Type of the ArcGIS Server (e. g. *Intranet* or *Internet*)
* All used datasources with the following details:
  * Type of the datasource  (e. g. *SDE* or *File Geodatabase*)
  * Name of the datasource (e. g. name of the Oracle or SQL Server instance)
  * Name of the dataset (name of the Feature Class, table, file, etc.)
  * Authentication mode (e. g. *Operating System Authentication* or *Database Authentication*)
  * Username used to access the datasource (only in case of an Enterprise Geodatabase)

Finally, all the information are exported with `HTTP POST` requests through the Elasticsearch REST API into the search index.

To ensure that the search index is always up-to-date, the FME Workspace has to be executed regularly, e. g. with a batch job or [FME Server](https://www.safe.com/fme/fme-server/).

The client communicates with the search index through the Elasticsearch REST API and executes a search. The search results are processed and displayed. The processing includes marking the datasets matching the search term so that they can be highlighted/colourized.


## Security
The code contains no mechanism to prohibit a malicious modification of the search index. If you do not protect the Elasticsearch backend, everybody knowing the URL of your backend can edit your search index via `HTTP PUT/POST/DELETE`. You have to handle this on your own, e. g. by using API keys (like [Searchly](http://docs.searchly.com) does) or through the web server.

## Working with the code
The code has been developed with Visual Studio 2015.

For local development, the client can be hosted in IIS Express. In order to work with a local backend, the following things are required:
1. Local instance of Elasticsearch must be installed and running
2. Elasticsearch index must have been created (e. g. with the FME Workspace `FME\MapServiceSearch_CreateIndex.fmw`)

After this, you have to set the Elasticsearch URL (either of the local backend or a hosted solution) in `MapServiceSearch.Web.App\script.js`:

    MapServiceSearchApp.service("client", function (esFactory) {
        return esFactory({
            host: "<<your URL goes here>>"
        });
    });

## Deployment
Since the AngularJS app is embedded into an ASP.NET Web Application, the publish mechanism of this project type can be used for deployment. The file `Deployment\Deployment.targets` contains the two MSBuild-Targets `DeployServer` and `DeployClient`. If you use a CI server, you can easily call them for an automated deployment:
* Client

		msbuild Deployment\Deployment.targets /t:DeployClient
* Backend

		msbuild Deployment\Deployment.targets /t:DeployServer

The destination path of the client is managed within the publish profile `MapServiceSearch.Web.App\Properties\PublishProfiles\Production.pubxml`. The MSBuild-Target copies all files belonging to the client to the destination path.

The MSBuild-Target to deploy the backend assumes that there is an on-premise installation of Elasticsearch on the same server where the web application is hosted. Since Elasticsearch typically runs under http://localhost:9200, you need a Reverse Proxy to expose the REST API of Elasticsearch. When using IIS, this can be easily accomplished by using the IIS module [URL Rewrite](https://www.iis.net/downloads/microsoft/url-rewrite). The file `api\web.config` contains an URL rewrite rule redirecting all traffic to http://localhost:9200. The MSBuild-Target simply copies this file to the destination path specified in `Deployment\Deployment.targets` and now IIS treats the folder `api` as an application, uses `web.config` with the URL rewrite rule and acts as a Reverse Proxy.  
