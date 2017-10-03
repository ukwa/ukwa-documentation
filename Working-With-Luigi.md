Note resolution of TMP location issue, and note this point and that requires() is a worse place for dynamic dependencies than run(), but run() serialises execution.

- Describe issues around atomic behaviour and suitable patterns.
- Explain our patterns of use.
- Thoroughly document the MoveToHdfs process.


https://luigi.readthedocs.io/en/stable/

How Luigi is deployed

/etc/luigi/client.cfg
/etc/luigi/logging.cfg

/etc/supervisor.d/luigi.ini

etc.