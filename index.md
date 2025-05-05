---
title: Automatisieren von Azure-Auslastungstests mit GitHub-Aktionen
permalink: index.html
layout: home
---

Die folgenden Übungen sollen Ihnen eine praktische Lernerfahrung bei der Implementierung von GitHub-Aktionen und -Workflows zur Automatisierung der Durchführung von Auslastungstests mit Azure Load Testing bereitstellen. 

## Übungen
<hr/>


{% assign labs = site.pages | where_exp:"page", "page.url contains '/Instructions/Labs'" %} {% for activity in labs  %}
* [{{ activity.lab.title }}]({{ site.github.url }}{{ activity.url }}) <br/> {{ activity.lab.description }} {% endfor %}
