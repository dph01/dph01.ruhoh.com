---
layout: page
title:  code test
date: 2012-06-04
tags: [scala, lift]
categories: [code]
---

      def create = {
        var event = eventRV.is                             
        "#hidden" #> SHtml.hidden(() => eventRV(event) ) & 
        "#eventname" #> SHtml.text(eventRV.is.eventName,   
                    name => eventRV.is.eventName(name) ) & 
        "#submit" #> SHtml.onSubmitUnit(processSubmit)     
      }
      
<pre class="pretty-print lang-scala">
def create = {
  var event = eventRV.is                             
  "#hidden" #> SHtml.hidden(() => eventRV(event) ) & 
  "#eventname" #> SHtml.text(eventRV.is.eventName,   
                  name => eventRV.is.eventName(name) ) & 
  "#submit" #> SHtml.onSubmitUnit(processSubmit)     
}
</pre>
