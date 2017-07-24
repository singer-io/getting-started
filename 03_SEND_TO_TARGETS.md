# 🎯 All about TARGETS 🎯

## Targets are very similar to TAPS in that they still adhere to the Singer spec. 

Right now there are targets made to send to a csv, Google Sheets, Magento, or Stitch but the possibilities are endless. To send your tap to a target here is an example using Google Sheets:


```bash
› tap-ip | target-gsheet -c config.json
```

Alternatively you could send it to a csv just as easy by doing this:

```bash
› tap-ip | target-csv -c config.json
```

To summarize the formula for pulling with a tap and sending to a target is:

```bash
› TAP-NAME -c TAP_CONFIG_FILE_HERE.json | TARGET-NAME -c TARGET_CONFIG_FILE_HERE.json
```

You might not always need config files, in which case it would just be:

```bash
› TAP-NAME | TARGET-NAME 
```
See? Easy. 

## If you'd like to create your own TARGET just follow the same tutorial for making a tap

just like building a tap. 
you consume the messages that a tap outputs and determine what to do with it / where to send it

prior to it being package up, in your dev environment you would run it like this
```bash
› TAP-NAME -c TAP_CONFIG_FILE_HERE.json | python my_target.py -c TARGET_CONFIG_FILE_HERE.json
```

once both bundled as packages then you can install via pip your other fav package system & then its this;
ou might not always need config files, in which case it would just be:

```bash
› TAP-NAME | TARGET-NAME 
```


