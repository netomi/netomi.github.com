---
layout: page
title: projects
permalink: /projects/
order: 2
---

<div class="projectFilter" style="text-align: center;">
  <a href="#" filter="*" class="current">All</a>
  <a href="#" filter=".ai">Artificial Intelligence</a>
  <a href="#" filter=".cv">Computer Vision</a>
  <a href="#" filter=".sdc">Self-Driving Cars</a>
  <a href="#" filter=".ios">iOS</a>
  <a href="#" filter=".misc">Misc</a>
</div>

<br/>

<div class="projectContainer">

{% for project in site.projects %}

{% if project.redirect %}
<div class="project {{ project.category }}">
    <div class="thumbnail">
        <a href="{{ project.redirect }}" target="_blank">
        {% if project.img %}
        <img class="thumbnail" src="{{ project.img }}"/>
        {% else %}
        <div class="thumbnail blankbox"></div>
        {% endif %}    
        <span>
            <h1>{{ project.title }}</h1>
            <br/>
            <p>{{ project.description }}</p>
        </span>
        </a>
    </div>
</div>
{% else %}

<div class="project {{ project.category }}">
    <div class="thumbnail">
        <a href="{{ site.baseurl }}{{ project.url }}">
        {% if project.img %}
        <img class="thumbnail" src="{{ project.img }}"/>
        {% else %}
        <div class="thumbnail blankbox"></div>
        {% endif %}    
        <span>
            <h1>{{ project.title }}</h1>
            <br/>
            <p>{{ project.description }}</p>
        </span>
        </a>

    </div>

    <span>
        <h1>{{ project.title }}</h1>
        <br/>
        <p>{{ project.description }}</p>
    </span>
</div>


{% endif %}

{% endfor %}

</div>
