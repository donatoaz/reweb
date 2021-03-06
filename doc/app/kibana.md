# Overview

Works nicely once warm, for viewing data.

Lambda cold start takes <= 10 seconds.
The first cold start (first communication with an Elastic) seems to time out once or twice, but after that, Kibana runs smoothly.

Simple usage clocks in at ~420 MB RAM usage per request, but I've given it 4 GB of RAM for more CPU speed.

⚠️ Any Kibana background threads will *not* work reliably, as the Lambda execution environment is halted when it is not processing
any request. This might impact Node Discovery, for example. It has only been tested with a trivial single-node Elastic deployment so far.

# Environment

Configure the following environment variables for the Lamba function:

| Environment variable | Value |
| --- | --- |
| ELASTICSEARCH_HOSTS | http://172.31.28.139:9200 |
| LOGGING_SILENT | true |
| NODE_OPTIONS | --max-old-space-size=3072 |
| XPACK_ENCRYPTEDSAVEDOJECTS_ENCRYPTIONKEY | X1234567890123456789012345678901234567890 |
| XPACK_SECURITY_ENCRYPTIONKEY | X1234567890123456789012345678901234567890 |

Adjust for your configuration as appropriate.

I have no experience with Kibana/Elastic, but my current understanding is that those crypto keys are used primarily for sessions (all Kibanas need
the same Key to access shared session data). It's just an arbitrary string, see
[Kibana docs](https://www.elastic.co/guide/en/kibana/current/security-settings-kb.html#security-session-and-cookie-settings).

# Dockerfile

```
FROM public.ecr.aws/apparentorder/reweb as reweb

FROM docker.elastic.co/kibana/kibana:7.10.2
COPY --from=reweb /reweb /reweb

ENV PATH_DATA /tmp  

ENV REWEB_APPLICATION_EXEC /usr/local/bin/dumb-init -- /usr/local/bin/kibana-docker
ENV REWEB_APPLICATION_PORT 5601
ENV REWEB_WAIT_CODE 302
ENV REWEB_WAIT_PATH /

ENTRYPOINT ["/reweb"]
```

#### Rationale

- Kibana needs to write some data, but it doesn't have to persist -- therefore `/tmp` is fine
- During startup, Kibana will accept HTTP requests, but will answer them with `503 Service Unavailable`, therefore re:Web needs to wait for a HTTP 302 first

# References

- https://www.elastic.co/guide/en/kibana/current/production.html#load-balancing-kibana
- https://www.elastic.co/guide/en/kibana/current/security-settings-kb.html#security-session-and-cookie-settings
