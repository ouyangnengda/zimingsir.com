## zimingsir.com

## 杂谈

[第一篇博客](https://zimingsir.com/talk/第一篇博客.html)

测试代码

<ul>

{% for post in site.posts %}

    <li>

      <h2><a href="{{ post.url }}">{{ post.title }}</a></h2>

      <p>{{ post.excerpt }}</p>

    </li>

{% endfor %}

</ul>


### 唉唉哎

<ul>

{% for post in site.love %}

    <li>

      <h2><a href="{{ post.url }}">{{ post.title }}</a></h2>

    </li>

{% endfor %}

</ul>

### fen ge

<ul>

{% for post in site.love %}

    <li>

      <h2><a href="{{ post.url }}">{{ post.title }}</a></h2>

      <p>{{ post.excerpt }}</p>

    </li>

{% endfor %}

</ul>

### asdf

<ul>

{% for post in site.categories[love] %}

    <li>

      <h2><a href="{{ post.url }}">{{ post.title }}</a></h2>

      <p>{{ post.excerpt }}</p>

    </li>

{% endfor %}

</ul>

    { % for post in site.categories.talk %}
    <li class="j-row j-list-i">
        { {post}}  //输出文章
    </li>
    { % endfor %}


    <!-- exampole -->
    <ul class="j-container">
        { % for post in site.posts %}
        <li class="j-row j-list-i">
            <h2 class="j-i-title"><a href="{ {site.baseurl}}{ {post.url}}">{ {post.title}}</a></h2>
            <p class="j-i-time">{ {post.date | date_to_string}}</p>
            <p class="j-i-tags">
                { %for tag in post.tags %}
                <span class="j-i-tag">{ {tag}}</span>
                { % endfor %}
            </p>
            <div class="j-container">
                <div class="j-row j-artclie-txt">{ {post.excerpt}}</div>
            </div>
        </li>
        { % endfor %}
    </ul>

