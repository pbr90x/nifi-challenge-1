# nifi-challenge-1
Apache NiFi Challenge - basic intro to Apache NiFi

1.  On your laptop or favorite Cloud environment, install a Kubernetes environment such as Minikube.
2.  On a container in the environment, configure NiFi.
3.  As the data ingest source for NiFi, use publicly available NASA Web/HTTP Server logs. 
4.  Show a Transform on the data such as counting the number of HTTP-GETs and store the transformed data locally on your file system.

Dataset:
http://ita.ee.lbl.gov/html/contrib/NASA-HTTP.html
ftp://ita.ee.lbl.gov/traces/NASA_access_log_Jul95.gz
ftp://ita.ee.lbl.gov/traces/NASA_access_log_Aug95.gz

First the pipeline obtains the two example NASA web logs from their FTP site using the GetFTP Processor. I use the File Filter Regex Property to only grab the two files I want for my pipeline. I also had to configure a few other properties on GetFTP (Username, Remote Path). I also disabled Delete Original to stop the processor from trying and failing to delete the source files of NASA FTP site each polling interval ;)

For testing and debugging purposes there is also a GetFile Processor configured as an input that will read files from ./data-in and feed them through the rest of the pipeline.

Next the IdentifyMimeType Processor is used to identify these as gzip files and add the appropriate MIME type attribute to the FlowFile.

The FlowFile is fed to the CompressContext Processor which looks at the MIME type attribute added by the IdentifyMimeType Processor, sees that they are gzip compressed and uncompresses them.

The FlowFile is passed to the RouteText Processor. Here I use a regular expression to match well-formed web logs for GET requests. The "Route to 'matched' if line matches all conditions" Routing Strategy is used. Matched lines are routed to the next step in the pipeline: CountText. For testing and debugging purposes, I route unmatched lines to a PutFile Processor that writes the unmatched content to ./data-unmatched. 

The CountText Processor simply counts the lines in the FlowFile. At this stage in the pipeline, only well-formed web logs for HTTP GET requests are present in the FlowFile. A line count represents the number of GET requests in the original input file. It is added as an FlowFile attribute named 'text.line.count'. For testing and debugging purposes I also have success output to LogAttribute which logs the FlowFile attributes to the nifi-app.log file locally.

The FlowFile with 'text.line.count' is now routed to AttributesJSON which replaces the contents of the FlowFile (a bunch of GET web logs) with a JSON representation of the FlowFile attributes including 'text.line.count'. There seems to be a bug where the core FlowFile attributes are still not included in the JSON despite the Include Core Attributes property being set to true on the Processor.

The re-written FileFile is now passed to PutFile and written out to the filesystem in ./data-out. The filename will be the same as the original input file. The content of the file will contain a value for 'text.line.count' which is the number of logs in the input file for HTTP GET requests.

I could improve on this by renaming the 'text.line.count' attribute to something like 'http.method.get.count', probably using the UpdateAttribute Processor. I would also like to figure out how to include the core FlowFile attributes in the AttributesJSON output (bug in that Processor?).

An enhancement would be to dynamically identify each distinct HTTP method present in the input logs files and then provide dynamic FlowFile attributes that are counts of the occurences of each one.

