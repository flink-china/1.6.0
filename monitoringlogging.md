<div class="col-lg-9 content" id="contentcol">
          

          





  
  
    
    
      
    
  

  
  
    
    
      



<ol class="breadcrumb">

  
  
    <li><i class="fa fa-bug title maindish" aria-hidden="true"></i> 调试 &amp; 监控</li>
  

  
  
    <li class="active">日志</li>
  

</ol>

<h1 id="how-to-use-logging">如何使用日志<a class="anchorjs-link " href="#how-to-use-logging" aria-label="Anchor link for: how to use logging" data-anchorjs-icon="" style="font-family: anchorjs-icons; font-style: normal; font-variant: normal; font-weight: normal; line-height: 1; padding-left: 0.375em;"></a></h1>




<p>Flink中的日志记录是使用slf4j日志记录接口实现的。Flink使用log4j作为底层日志记录框。我们还提供了logback配置文件，并将它们作为属性传递给JVM。愿意使用logback来代替log4j的用户可以排除掉log4j(或者从lib文件夹中删除它)。</p>

<ul id="markdown-toc">
  <li><a href="#configuring-log4j" id="markdown-toc-configuring-log4j">配置 Log4j</a></li>
  <li><a href="#configuring-logback" id="markdown-toc-configuring-logback">配置logback</a></li>
  <li><a href="#best-practices-for-developers" id="markdown-toc-best-practices-for-developers">开发人员的最佳实践</a></li>
</ul>

<h2 id="configuring-log4j">配置 Log4j<a class="anchorjs-link " href="#configuring-log4j" aria-label="Anchor link for: configuring log4j" data-anchorjs-icon="" style="font-family: anchorjs-icons; font-style: normal; font-variant: normal; font-weight: normal; line-height: 1; padding-left: 0.375em;"></a></h2>

<p>Log4j是使用属性文件控制的。 在Flink中, 这个文件通常被称为 <code class="highlighter-rouge">log4j.properties</code>。 我们通过 <code class="highlighter-rouge">-Dlog4j.configuration=</code> 参数将文件名和文件的位置传递给JVM。</p>

<p>Flink附带以下默认属性文件:</p>

<ul>
  <li><code class="highlighter-rouge">log4j-cli.properties</code>: 由Flink命令行客户端使用 (e.g. <code class="highlighter-rouge">flink run</code>) (不是在集群上执行的代码)</li>
  <li><code class="highlighter-rouge">log4j-yarn-session.properties</code>: 当打开一个YARN session的时候，由Flink命令行客户端使用 (<code class="highlighter-rouge">yarn-session.sh</code>)</li>
  <li><code class="highlighter-rouge">log4j.properties</code>: JobManager/Taskmanager的日志 (both standalone and YARN)</li>
</ul>

<h2 id="configuring-logback">配置 logback<a class="anchorjs-link " href="#configuring-logback" aria-label="Anchor link for: configuring logback" data-anchorjs-icon="" style="font-family: anchorjs-icons; font-style: normal; font-variant: normal; font-weight: normal; line-height: 1; padding-left: 0.375em;"></a></h2>

<p>对于用户和开发人员来说，控制日志框架非常重要。日志框架的配置完全由配置文件完成。配置文件必须通过设置环境变量 <code class="highlighter-rouge">-Dlogback.configurationFile=&lt;file&gt;</code> 或者在classpath中设置 <code class="highlighter-rouge">logback.xml</code> 来指定。 <code class="highlighter-rouge"></code>conf文件夹中包含了一个可修改的<code class="highlighter-rouge">logback.xml</code> 文件，如果Flink在IDE外启动并提供了启动脚本，则使用该配置文件。
提供的 <code class="highlighter-rouge">logback.xml</code>文件的形式如下：</p>

<figure class="highlight"><pre><code class="language-xml" data-lang="xml"><span class="nt">&lt;configuration&gt;</span>
    <span class="nt">&lt;appender</span> <span class="na">name=</span><span class="s">"file"</span> <span class="na">class=</span><span class="s">"ch.qos.logback.core.FileAppender"</span><span class="nt">&gt;</span>
        <span class="nt">&lt;file&gt;</span>${log.file}<span class="nt">&lt;/file&gt;</span>
        <span class="nt">&lt;append&gt;</span>false<span class="nt">&lt;/append&gt;</span>
        <span class="nt">&lt;encoder&gt;</span>
            <span class="nt">&lt;pattern&gt;</span>%d{HH:mm:ss.SSS} [%thread] %-5level %logger{60} %X{sourceThread} - %msg%n<span class="nt">&lt;/pattern&gt;</span>
        <span class="nt">&lt;/encoder&gt;</span>
    <span class="nt">&lt;/appender&gt;</span>

    <span class="nt">&lt;root</span> <span class="na">level=</span><span class="s">"INFO"</span><span class="nt">&gt;</span>
        <span class="nt">&lt;appender-ref</span> <span class="na">ref=</span><span class="s">"file"</span><span class="nt">/&gt;</span>
    <span class="nt">&lt;/root&gt;</span>
<span class="nt">&lt;/configuration&gt;</span></code></pre></figure>

<p>例如，如果想要控制 <code class="highlighter-rouge">org.apache.flink.runtime.jobgraph.JobGraph</code>的日志等级，就必须添加如下语句到配置文件中</p>

<figure class="highlight"><pre><code class="language-xml" data-lang="xml"><span class="nt">&lt;logger</span> <span class="na">name=</span><span class="s">"org.apache.flink.runtime.jobgraph.JobGraph"</span> <span class="na">level=</span><span class="s">"DEBUG"</span><span class="nt">/&gt;</span></code></pre></figure>

<p>更多的logback的配置信息可以看<a href="http://logback.qos.ch/manual/configuration.html">LOGback’s manual</a>.</p>

<h2 id="best-practices-for-developers">开发人员的最佳实践<a class="anchorjs-link " href="#best-practices-for-developers" aria-label="Anchor link for: best practices for developers" data-anchorjs-icon="" style="font-family: anchorjs-icons; font-style: normal; font-variant: normal; font-weight: normal; line-height: 1; padding-left: 0.375em;"></a></h2>

<p>使用slf4j的日志记录器是通过调用创建的</p>

<figure class="highlight"><pre><code class="language-java" data-lang="java"><span class="kn">import</span> <span class="nn">org.slf4j.LoggerFactory</span>
<span class="kn">import</span> <span class="nn">org.slf4j.Logger</span>

<span class="n">Logger</span> <span class="n">LOG</span> <span class="o">=</span> <span class="n">LoggerFactory</span><span class="o">.</span><span class="na">getLogger</span><span class="o">(</span><span class="n">Foobar</span><span class="o">.</span><span class="na">class</span><span class="o">)</span></code></pre></figure>

<p>为了从slf4j中获得最大利益，建议使用它的占位符机制。
使用占位符可以避免不必要的字符串结构，以防日志记录级别设置得太高以致消息无法记录。
占位符的语法如下:</p>

<figure class="highlight"><pre><code class="language-java" data-lang="java"><span class="n">LOG</span><span class="o">.</span><span class="na">info</span><span class="o">(</span><span class="s">"This message contains {} placeholders. {}"</span><span class="o">,</span> <span class="mi">2</span><span class="o">,</span> <span class="s">"Yippie"</span><span class="o">);</span></code></pre></figure>

<p>占位符还可以与需要记录的异常一起使用。</p>

<figure class="highlight"><pre><code class="language-java" data-lang="java"><span class="k">catch</span><span class="o">(</span><span class="n">Exception</span> <span class="n">exception</span><span class="o">){</span>
	<span class="n">LOG</span><span class="o">.</span><span class="na">error</span><span class="o">(</span><span class="s">"An {} occurred."</span><span class="o">,</span> <span class="s">"error"</span><span class="o">,</span> <span class="n">exception</span><span class="o">);</span>
<span class="o">}</span></code></pre></figure>

<p><a href="#top" class="top pull-right"><span class="glyphicon glyphicon-chevron-up"></span> Back to top</a></p>


        </div>