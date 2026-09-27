---
layout: page
title: 关于
permalink: /about/
---

<script>
/* src/04 原样移植：终端逐行 + 技能条 + 经历节点 + 入场编排 */
window.RISE_TARGETS=['.title','.cols .sec','.cols .skill','.cols .note','.cols .contact span'];
window.addEventListener('load',()=>{
  document.querySelectorAll('.term .ln').forEach((el,i)=>setTimeout(()=>el.classList.add('show'),250+i*330));
  document.querySelectorAll('.fill').forEach(el=>{el.style.setProperty('--w',el.style.width);setTimeout(()=>el.classList.add('go'),800);});
  document.querySelectorAll('.job').forEach((el,i)=>setTimeout(()=>el.classList.add('in'),600+i*140));
});
</script>

<div class="title">{{ site.data.about.title }}</div>

<div class="term"><div class="bar"><i class="r"></i><i class="y"></i><i class="g"></i><span>{{ site.data.about.terminal.prompt }}</span></div>
<span class="ln"><span class="p">$</span> whoami<br>
{{ site.data.about.terminal.whoami }}</span>
<span class="ln"><span class="p">$</span> cat now.txt<br>
<span class="g">"{{ site.data.about.terminal.cat_now_txt.quote }}"</span> <span class="gr">{{ site.data.about.terminal.cat_now_txt.comment }}</span></span>
<span class="ln"><span class="p">$</span> uptime<br>
<span class="b">{{ site.data.about.terminal.uptime_highlight }}</span> {{ site.data.about.terminal.uptime_detail }}</span>
<span class="ln"><span class="p">$</span> <span class="gr">█</span></span>
</div>

<div class="cols">
  <div>
    <h3 class="sec">cat experience.log</h3>
    <div class="exp">
      {% for job in site.data.about.experience %}
      <div class="job"><span class="when">{{ job.period }}</span><h4>{{ job.title }} <em>/ {{ job.track }}</em></h4><p>{{ job.summary }}</p></div>
      {% endfor %}
    </div>
  </div>
  <div>
    <h3 class="sec">cat skills.conf</h3>
    {% for s in site.data.about.skills %}
    <div class="skill"><div class="nm"><span>{{ s.name }}</span><span>{{ s.level }}</span></div><div class="track"><div class="fill" style="width:{{ s.track }}%"></div></div></div>
    {% endfor %}
    <div class="note">{{ site.data.about.note }}</div>
    <h3 class="sec">ping contact</h3>
    <div class="contact">{% for c in site.data.about.contact %}<span>{{ c }}</span>{% endfor %}</div>
  </div>
</div>