# 🎯 All about TARGETS 🎯

## Targets are very similar to TAPS in that they still adhere to the JSON Schema standard. 

Right now there are targets made to send to a csv, Google Sheets, Magento, or Stitch but the possibilities are endless. To send your tap to a target here is an example using Google Sheets:


```bash
› python tap_ip.py | target-gsheet -c config.json
```

Alternatively you could send it to a csv just as easy by doing this:

```bash
› python tap_ip.py | target-csv -c config.json
```

To summarize the formula for pulling with a tap and sending to a target is:

```bash
› python YOUR_TAP_FILE.py -c TAP_CONFIG_FILE_HERE.json | TARGET-TYPE -c TARGET_CONFIG_FILE_HERE.json
```

You might not always need config files, in which case it would just be:

```bash
› python YOUR_TAP_FILE.py | TARGET-TYPE 
```
See? Easy. 

## If you'd like to create your own TARGET just follow the same tutorial for making a tap
