---
layout: page
title: projects
permalink: /projects/
order: 2
---

<div class="projectFilter" style="text-align: center;">
  <a href="#" filter="*" class="current">All</a>
  <a href="#" filter=".bytecode">Bytecode engineering</a>
  <a href="#" filter=".scientific">Scientific computing</a>
  <a href="#" filter=".game">Game theory</a>
  <a href="#" filter=".misc">Misc</a>
</div>

<br/>

<div class="projectContainer">

{% for project in site.projects %}

<div class="project {{ project.category }}">
    <div class="thumbnail">
        <a href="{{ site.baseurl }}{{ project.url }}">
        {% if project.img %}
        <img class="thumbnail" src="{{ project.img }}"/>
        {% else %}
        <div class="thumbnail blankbox"></div>
        {% endif %}    
        </a>

    </div>

    <div class="description">
        <span>
        <h1>{{ project.title }}</h1>
        <br/>
        <p>{{ project.description }}</p>

        <ul class="left">        
        {% for tag in project.tags %}

        <li>{{ tag | replace:'_',' ' }}</li>
        
        {% endfor %}
        </ul>

        </span>
    </div>
</div>

{% endfor %}

</div>
