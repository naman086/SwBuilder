#!/usr/bin/python
#
# Static Website Builder
# A simple and lightweight static website building script based on SCons.
# Author: Donghao Ren

# This program is free software: you can redistribute it and/or modify
# it under the terms of the GNU General Public License as published by
# the Free Software Foundation, either version 3 of the License, or
# (at your option) any later version.

# This program is distributed in the hope that it will be useful,
# but WITHOUT ANY WARRANTY; without even the implied warranty of
# MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
# GNU General Public License for more details.

# You should have received a copy of the GNU General Public License
#  along with this program.  If not, see <http://www.gnu.org/licenses/>.

import re
import os
import hashlib
import subprocess
import json
import yaml
import pystache
import pytz
import copy

from hashlib import sha256
from datetime import datetime

# Minify JS and CSS or not.
option_minify = ARGUMENTS.get('minify', 'true')
option_striphtml = ARGUMENTS.get('striphtml', 'true')
option_display_timezone = "Asia/Shanghai"
option_time_format = "%b %d, %Y, %I:%M %p, %Z"

# Current time.
build_time = pytz.timezone("UTC").localize(datetime.utcnow()).astimezone(pytz.timezone(option_display_timezone))

# Global Meta strings, can be used in files with syntax {{key}}
# Only 0-9a-zA-Z and -, _, . are allowed for key.
meta_strings = {
    'build_time'     : build_time.strftime(option_time_format),
    'build_time_iso' : build_time.isoformat()
}

# Global variables.
output_directory = "deploy"
temporary_directory = "temp"

if not os.path.exists(temporary_directory):
    os.makedirs(temporary_directory)

if not os.path.exists(output_directory):
    os.makedirs(output_directory)
    
# Delimiters
delim_L = "{{"
delim_R = "}}"

def Delimiter(L, R):
    global delim_L
    global delim_R
    global regex_partial, regex_include, regex_js, regex_css, regex_ref, regex_active, regex_meta;
    global regex_mustache, regex_mustache_render, regex_mustache_render_first
    global regex_mustache_local
    global regex_mustache_render_yaml_include
    global regex_hash
    
    global regex_yaml_info, regex_yaml_info_htcomment
    
    # Set
    delim_L = L
    delim_R = R
    
    reg_L = re.escape(delim_L)
    reg_R = re.escape(delim_R)
    
    htcomment_start = re.escape("<!--")
    htcomment_end = re.escape("-->")
    
    regex_partial = re.compile(reg_L + r' *partial: *([0-9a-zA-Z\-\_\.]+) *' + reg_R)
    regex_include = re.compile(reg_L + r' *include: *([0-9a-zA-Z\-\_\.]+) *' + reg_R)
    
    regex_js = re.compile(reg_L + r" *js: *([0-9a-zA-Z\-\_\.\,\/]+) *" + reg_R)
    regex_css = re.compile(reg_L + r" *css: *([0-9a-zA-Z\-\_\.\,\/]+) *" + reg_R)
    regex_ref = re.compile(reg_L + r" *ref: *([0-9a-zA-Z\-\_\.\,\/]+) *" + reg_R)
    regex_hash = re.compile(reg_L + r" *hash: *([0-9a-zA-Z\-\_\.\,\/]+) *" + reg_R)
    
    regex_active = re.compile(reg_L + r" *(active|current|active-this|current-this): *([0-9a-zA-Z\-\_\.\,\/]+) *" + reg_R)
    regex_meta = re.compile(reg_L + r" *([0-9a-zA-Z\_\-\.]+) *" + reg_R)
    
    regex_mustache = re.compile(htcomment_start + r" *mustache +([0-9a-zA-Z\-\_\.]+) *\{ *" + htcomment_end
                                + r"(.*?)" +
                                htcomment_start + r" *\} *mustache *" + htcomment_end, re.DOTALL)

    regex_mustache_render = re.compile(htcomment_start + r" *mustache\-render +([0-9a-zA-Z\-\_\.]+) *\{ * " + htcomment_end
                                + r"(.*?)" +
                                htcomment_start + r" *\} *mustache\-render *" + htcomment_end, re.DOTALL)
    
    regex_mustache_local = re.compile(htcomment_start + r" *mustache\-local *\{ *" + htcomment_end
                                + r"(.*?)" +
                                htcomment_start + r" *\} *mustache\-local *" + htcomment_end, re.DOTALL)

    regex_mustache_render_first = re.compile(htcomment_start + r" *mustache\-render +([0-9a-zA-Z\-\_\.]+) *\{ * " + htcomment_end)
    regex_mustache_render_yaml_include = re.compile(reg_L + r" *yaml-include *: *([0-9a-zA-Z\ \_\-\.]+)" + reg_R)
    
    regex_yaml_info = re.compile(r"(\#\|\-{4}\-+)"
                               + r"(.*?)"
                               + r"\1", re.DOTALL)
    regex_yaml_info_htcomment = re.compile(r"(\<\!\-\-[\-]*\{)"
                               + r"(.*?)"
                               + r"\}\-\-[\-]*\>", re.DOTALL)

def mustache_render(templ, obj):
    parsed = pystache.parse(ensure_unicode(templ, "utf-8"), delimiters = (u"[[", u"]]"))
    return pystache.render(parsed, obj)

# Ensure that the text read from a file is in unicode.
def ensure_unicode(text, encoding):
    try:
      return unicode(text, encoding)
    except TypeError:
      return text

Delimiter("{{", "}}")

# Resolve include files.
# soruce: scons file object.
def resolve_includes(source):
    global regex_include
    data = ensure_unicode(source.get_text_contents(), 'utf-8')
    data = regex_include.sub(lambda m: resolve_includes(File(m.group(1))), data)
    return data

def include_build_function(target, source, env, minify = '', mustache = 0):
    data = "";
    for s in source:
        data += resolve_includes(s)

    for t in target:
        fn = str(t)
        if minify == 'js' or minify == 'css':
            p = subprocess.Popen(["java", "-jar", "utils/yuicompressor.jar", "--type", minify, "-o", fn], stdin = subprocess.PIPE, stdout = None, stderr = subprocess.STDOUT)
            p.stdin.write(data.encode('utf-8'))
            p.stdin.close()
            r = p.wait()
            if r != 0:
                return Exception("Failed to build %s" % fn)
        else:
            if mustache:
                mustache_templates = { };
                
                def yaml_include_f(m):
                    k = m.group(1).split(" ")
                    fn = k[0]
                    key = k[1]
                    obj = yaml.load(read_yaml_file(fn))
                    dict = { };
                    dict[key] = obj
                    return yaml.dump(dict);
                
                data = regex_mustache_render_yaml_include.sub(lambda m: yaml_include_f(m), data)
                    
                def mustache_renderf(m):
                    if m.group(1) in mustache_templates:
                        templ = mustache_templates[m.group(1)]
                    else:
                        templ = m.group(1).split(".")
                        mustache = read_mustache_file(templ[0])
                        def mustache_subf(m):
                            mustache_templates[templ[0] + "." + m.group(1)] = m.group(2)
                            return ""
                        regex_mustache.sub(lambda m: mustache_subf(m), mustache)
                        templ = mustache_templates[m.group(1)]
                    obj = yaml.load(m.group(2))
                    return mustache_render(templ, obj)
                data = regex_mustache_render.sub(lambda m: mustache_renderf(m), data);
            f = open(str(fn), 'w')
            f.write(data.encode('utf-8'))
            f.close()
    return None

def inc_build_function(target, source, env):
    return include_build_function(target, source, env)

def inc_build_function_mustache(target, source, env):
    return include_build_function(target, source, env, mustache = 1)

def js_build_function(target, source, env):
    return include_build_function(target, source, env, minify = 'js')

def css_build_function(target, source, env):
    return include_build_function(target, source, env, minify = 'css')

# JS and CSS builder.
if option_minify == 'true':
    js_builder = Builder(action = js_build_function)
    css_builder = Builder(action = css_build_function)
else:
    js_builder = Builder(action = inc_build_function)
    css_builder = Builder(action = inc_build_function)

include_builder = Builder(action = inc_build_function_mustache)

# Markdown builder.
markdown_builder = Builder(action = 'cat $SOURCES | markdown.php > $TARGET')

# BibTex2JSON builder.
BibTex2YAML_builder = Builder(action = 'python bibtex2yaml.py $SOURCES > $TARGET')

# Copy builder, just copy files.
copy_builder = Builder(action = 'cp $SOURCE $TARGET')

# Concat builder, concat source files to target files, a line-break is
# inserted between files.
def concat_build_function(target, source, env):
    data = "";
    for s in source:
        data += ensure_unicode(s.get_text_contents(), 'utf-8') + u"\n"
    
    for t in target:
        f = open(str(t), 'w')
        f.write(data.encode('utf-8'))
        f.close()
    return None

concat_builder = Builder(action = concat_build_function)

# YAML to JSON builder
def yaml2json_build_function(target, source, env):
    data = "";
    for s in source:
        data += ensure_unicode(s.get_text_contents(), 'utf-8') + u"\n"
    
    obj = yaml.load(data)
    
    data = json.dumps(obj)
    
    for t in target:
        f = open(str(t), 'w')
        f.write(data.encode('utf-8'))
        f.close()
    return None

yaml2json_builder = Builder(action = yaml2json_build_function)

# Scan for partials, make them dependencies.
def partial_scanner(node, env, path):
    files = [];
    text = ensure_unicode(node.get_text_contents(), 'utf-8')
    result = regex_partial.findall(text)
    for partial in result:
        files.append(get_partial_path(partial))
    return env.File(files);

# Scan for mustaches, make them dependencies.
def mustache_scanner(node, env, path):
    files = [];
    text = ensure_unicode(node.get_text_contents(), 'utf-8')
    result = regex_mustache_render_first.findall(text)
    for mustache in result:
        files.append(get_mustache_path(mustache.split(".")[0]))
    result = regex_mustache_render_yaml_include.findall(text)
    for yaml in result:
        files.append(get_yaml_path(yaml.split(" ")[0]))
    return env.File(files);    

# Include scanner, scan for included files.
def include_scanner(node, env, path, parents = []):
    files = [];
    text = ensure_unicode(node.get_text_contents(), 'utf-8')
    path = os.path.dirname(node.rstr())
    if path == "":
        path = "."
    result = regex_include.findall(text)
    for inc in result:
        if inc in parents:
            raise Exception("Circular includes on '%s'." % str(node))
        files.append(path + "/" + inc)
    r = env.File(files);
    for inc in result:
        r += include_scanner(File(inc), env, path, parents + [inc])
    return r

# Substitute builder, Template + Partials + HTML = Output Page.
def substitute_build_function(target, source, env):
    global regex_hash
    
    template = ensure_unicode(source[1].get_text_contents(), 'utf-8');
    content = ensure_unicode(source[0].get_text_contents(), 'utf-8');
    template = template.replace(delim_L + "content" + delim_R, content);
    
    def read_temp_hash_file(f):
        path = temporary_directory + "/" + f
        tf = open(path, 'r')
        data = tf.read().strip()
        tf.close()
        return data
    
    template = regex_hash.sub(lambda m: "{{hash: " + read_temp_hash_file(m.group(1)) + "}}", template)
    for t in target:
        f = open(str(t), 'w')
        f.write(template.encode('utf-8'))
        f.close()
    return None
    
substitute_builder = Builder(action = substitute_build_function)

def strip_html(path):
    if option_striphtml == 'true':
        if path.endswith(".html"):
            path = path[:-5]
        if path.endswith("index"):
            path = path[:-5]
        if path.endswith("/"):
            path = path[:-1]
        if path == "":
            path = "."
    return path

def active_page(this_page, target, str):
    pA = strip_html(this_page)
    pB = strip_html(target)
    if str.endswith("-this"):
        if pA == pB: return str[:-5]
        return ""        
    else:
        if(pA.startswith(pB)): return str
        return ""

def meta_enrich(yaml_data, local_meta):
    obj = yaml.load(yaml_data)
    for key in obj:
        local_meta[key] = obj[key]
    return ""

# HTML builder function.
# Change {{js: js_files}}, {{css: css_files}} to compiled locations.
def html_build_function(target, source, env):
    global regex_js, regex_css, regex_ref, regex_meta, regex_partial
    global regex_yaml_info, regex_yaml_info_htcomment
    global regex_mustache_local
    
    data = "";
    for s in source:
        data += ensure_unicode(s.get_text_contents(), 'utf-8')

    data = regex_partial.sub(lambda m: read_partial_file(m.group(1)), data)

    data = regex_meta.sub(lambda m: meta_substitute(m.group(1), m.group(0), meta_strings), data)
    
    local_meta = {
        'title': env['SWB_title'],
        'pageurl': env['SWB_url'],
        'extra': env['SWB_extra']
    }
    
    data = regex_yaml_info.sub(lambda m: meta_enrich(m.group(2), local_meta), data)
    data = regex_yaml_info_htcomment.sub(lambda m: meta_enrich(m.group(2), local_meta), data)
    
    if 'blog' in local_meta:
        append_blog_info(local_meta['blog'])
    
    # Local mustache first.
    data = regex_mustache_local.sub(lambda m: mustache_render(m.group(1), local_meta), data)
    # Then simple ones.
    data = regex_meta.sub(lambda m: meta_substitute(m.group(1), m.group(0), local_meta), data)

    for t in target:
        fn = str(t);

        html = regex_js.sub(lambda m: make_relative_path(fn, m.group(1)), data)
        html = regex_css.sub(lambda m: make_relative_path(fn, m.group(1)), html)
        html = regex_ref.sub(lambda m: strip_html(make_relative_path(fn, m.group(1))), html)
        html = regex_active.sub(lambda m: active_page(env['SWB_url'], m.group(2), m.group(1)), html)
                
        html = html.replace(delim_L + "L" + delim_R, delim_L)
        html = html.replace(delim_L + "R" + delim_R, delim_R)
        html = html.replace(delim_L + "-" + delim_R, "-")
        
        html = regex_hash.sub("", html)

        f = open(fn, 'w')
        f.write(html.encode('utf-8'))
        f.close()
    return None

html_builder = Builder(action = html_build_function)

# ImageMagick builder.
def imagemagick_generator(source, target, env, for_signature):
    return 'convert "%s" %s "%s"' % (source[0], env['SWB_args'], target[0])

imagemagick_builder = Builder(generator = imagemagick_generator)

# The SCons environment.
env = Environment(
    BUILDERS = {
        'Javascript' : js_builder,
        'CSS' : css_builder,
        'Concat' : concat_builder,
        'Copy' : copy_builder,
        'Markdown' : markdown_builder,
        'Substitute' : substitute_builder,
        'HTML' : html_builder,
        'ImageMagick' : imagemagick_builder,
        'ResolveIncludes' : include_builder,
        'BibTex2YAML': BibTex2YAML_builder,
        'YAML2JSON' : yaml2json_builder
    }
);

# Include scanner, scan for included files.
def blog_template_scanner(node, env, path, parents = []):
    files = ["%s/blog.hash" % temporary_directory];
    r = env.File(files);
    return r

def union_scanner(scanners):
    def rf(node, env, path):
        files = []
        for scanner in scanners:
            files = files + scanner(node, env, path)
        return files
    return rf

# Add our template scanner.
env.Append(SCANNERS = Scanner(function = blog_template_scanner, skeys = ['.blog_template']))
env.Append(SCANNERS = Scanner(function = partial_scanner, skeys = ['.compiled']))
env.Append(SCANNERS = Scanner(function = include_scanner, skeys = ['.js', '.css']))
env.Append(SCANNERS = Scanner(function = union_scanner([mustache_scanner, include_scanner, partial_scanner]),
                              skeys = ['.html', '.md']))
# Utility functions.
def get_file_extension(filename):
    return os.path.splitext(filename)[1][1:].strip().lower();

def get_partial_path(name):
    return "%s/%s.partial" % (temporary_directory, name)
    
def get_mustache_path(name):
    return "%s/%s.mustache" % (temporary_directory, name)
    
def get_yaml_path(name):
    return "%s/%s.yaml" % (temporary_directory, name)

def read_file(name):
    f = open(name, 'r')
    content = f.read()
    f.close()
    return content

def read_partial_file(name):
    temp = get_partial_path(name)
    f = open(temp, 'r')
    content = f.read()
    f.close()
    return content

def read_mustache_file(name):
    temp = get_mustache_path(name)
    f = open(temp, 'r')
    content = f.read()
    f.close()
    return content    
    
def read_yaml_file(name):
    temp = get_yaml_path(name)
    f = open(temp, 'r')
    content = f.read()
    f.close()
    return content
    
def get_temp_path(url):
    return "%s/%s.content" % (temporary_directory, url)
    
def content_html(target, source):
    extension = get_file_extension(source)
    if extension == "html":
        temp = "%s/%s.resolved" % (temporary_directory, source)
        env.ResolveIncludes(temp, source)
        env.Copy(target, temp)
    elif extension == "md":
        temp = "%s/%s.resolved" % (temporary_directory, source)
        env.ResolveIncludes(temp, source)
        env.Markdown(target, temp)

def make_relative_path(html_path, file_path):
    html_path = html_path.replace(output_directory + "/", "");
    return os.path.relpath(file_path, os.path.dirname(html_path));

def meta_substitute(meta_name, original, metadict):
    split = meta_name.split(".", 1)
    if split[0] in metadict:
        if len(split) == 1:
            return metadict[split[0]]
        else:
            return meta_substitute(split[1], original, metadict[split[0]])
    return original

# Set output directory.
def OutputDirectory(dir):
    global output_directory
    output_directory = dir

# Set temporary directory.
def TemporaryDirectory(dir):
    global temporary_directory
    temporary_directory = dir

# Add mustache.

def Mustache(name, source):
    target_name = get_mustache_path(name)
    env.Copy(target_name, source)

# Add partial.
def Partial(name, source):
    target_name = get_partial_path(name)
    content_html(target_name, source)

# Add page.
def Page(url, source, template, title = '', extra = {}):
    temp_name = get_temp_path(url)
    content_html(temp_name, source)
    temp = "%s/%s.compiled" % (temporary_directory, url)
    output = "%s/%s" % (output_directory, url)
    env.Substitute(temp, [ temp_name, template ])
    env.HTML(output, temp, SWB_title = title, SWB_url = url, SWB_extra = extra)
    
# Add pure HTML file, just expand metadata.
def HTML(url, source, title = '', extra = {}):
    output = "%s/%s" % (output_directory, url)
    extension = get_file_extension(source)
    if extension == 'html':
        temp = "%s/%s.resolved" % (temporary_directory, url)
        env.ResolveIncludes(temp, source)
        env.HTML(output, temp, SWB_title = title, SWB_url = url, SWB_extra = extra)
    else:
        temp_r = "%s/%s.resolved" % (temporary_directory, url)
        temp_c = "%s/%s.compiled" % (temporary_directory, url)
        env.ResolveIncludes(temp_r, source)
        env.Markdown(temp_c, temp_r)
        env.HTML(output, temp_c, SWB_title = title, SWB_url = url, SWB_extra = extra)

# Add Javascript, source can be multiple files.
def Javascript(url, source):
    mins = [];
    output = "%s/%s" % (output_directory, url)
    for s in source:
        min = "%s/%s.min" % (temporary_directory, s)
        env.Javascript(min, s)
        mins.append(min)
    env.Concat(output, mins)

# Add CSS, source can be multiple files.
def CSS(url, source):
    mins = [];
    output = "%s/%s" % (output_directory, url)
    for s in source:
        min = "%s/%s.min" % (temporary_directory, s)
        env.CSS(min, s)
        mins.append(min)
    env.Concat(output, mins)

def Binary(url, source):
    output = "%s/%s" % (output_directory, url)
    env.Copy(output, source)

def Binaries(url_path, list):
    for file in list:
        Binary(url_path + file, file)

def Image(url, source):
    Binary(url, source)

def ImageMagick(url, source, args = ""):
    output = "%s/%s" % (output_directory, url)
    env.ImageMagick(output, source, SWB_args = args)

def YAML2JSON(url, source):
    output = "%s/%s" % (output_directory, url)
    env.YAML2JSON(output, source)

def Images(url_path, list):
    Binaries(url_path, list)
    
def Find(pattern, directory = "."):
    dir = Dir(directory)
    r = dir.glob(pattern, strings=True)
    return map(lambda x: (x, directory + "/" + x), r)

def FindFiles(pattern, directory = "."):
    dir = Dir(directory)
    r = dir.glob(pattern, strings=True)
    return map(lambda x: directory + "/" + x, r)


def TargetList(f, url_path, list):
    for filename, path in list:
        f(url_path + "/" + filename, path)

def SourceList(f, list):
    for path in list:
        f(path)

# Meta strings.
def Meta(key, value):
    global meta_strings;
    meta_strings[key] = value

def BibTex(name, source):
    temp = "%s/%s.yaml" % (temporary_directory, name)
    env.BibTex2YAML(temp, source)
    
# Blog functionalities.
blog_articles = []
blog_tags = {}
blog_path = "blog"

blogdb = {}
blogdb_articles = []
blogdb_tags = []
blogdb_tags_map = { }
blogdb_tags_defined = []

blogdb_articles_list = {}
blogdb_tags_list = {}

def append_blog_info(meta):
    if 'permlink' in meta:
        permlink = meta['permlink']
        if not permlink in blogdb: return
        article = blogdb[permlink]
        for key in ['date_display', 'update_display', 'prev', 'next']:
            if key in article:
                meta[key] = article[key]
        if 'tags' in meta:
            meta['tags_comma'] = ", ".join(map(lambda m: blogdb_tags_map[m]['name'], meta['tags']))
            meta['tags'] = map(lambda m: { 'name': blogdb_tags_map[m]['name'], 'tag': m, 'path': blogdb_tags_map[m]['path'] }, meta['tags'])
            
    meta['blogdb_tags'] = blogdb_tags
    meta['blogdb_recent'] = map(lambda m: blogdb[m], blogdb_articles[0:10])

def blog_date_parse(s):
    date = datetime.strptime(" ".join(s.split(" ")[0:2]), "%Y-%m-%d %H:%M:%S")
    tz = pytz.timezone(s.split(" ")[2])
    return tz.localize(date)

# Add a post to the blog index, internal use only, don't call it outside.
def BlogPost(file):
    global blog_articles, blog_tags
    
    data = read_file(file)
    local_meta = {}
    global regex_yaml_info, regex_yaml_info_htcomment
    data = regex_yaml_info.sub(lambda m: meta_enrich(m.group(2), local_meta), data)
    data = regex_yaml_info_htcomment.sub(lambda m: meta_enrich(m.group(2), local_meta), data)

    # Make categories.
    blog = local_meta['blog']
    
    if 'status' in blog and blog['status'] == 'private':
        return
    
    blog['src'] = file
    blog['title'] = local_meta['title']
    if 'tags' in blog:
        for tag in blog['tags']:
            if not tag in blog_tags:
                blog_tags[tag] = { 'articles': [], 'name': tag }
            blog_tags[tag]['articles'].append(blog['permlink'])
    blog_articles.append(blog)
    return blog

def BlogTags(file):
    data = read_file(file)
    tags = yaml.load(data)
    for obj in tags:
        name = obj['name']
        tag = obj['key']
        blogdb_tags_defined.append(tag)
        if tag in blog_tags:
            blog_tags[tag]['name'] = name
        else:
            blog_tags[tag] = { 'name': name, 'articles': [] }

def blog_sort_articles(articles):
    global blogdb
    return sorted(articles, key = lambda m: blogdb[m]['date'], reverse = True)
    
def blog_build_list(articles):
    page_articles = 2
    N = len(articles)
    list = {
        'count': N,
        'pages': [],
        'articles': articles
    }
    N_pages, reminder = divmod(N, page_articles)
    if reminder > 0: N_pages += 1
    for pageindex in range(N_pages):
        list['pages'].append({
            'articles': articles[pageindex * page_articles:pageindex * page_articles + page_articles],
            'next': pageindex + 1 if pageindex < N_pages else None,
            'prev': pageindex - 1 if pageindex > 0 else None
        })
    return list

def BlogInit(path):
    blog_path = path

def blog_article_path(article):
    return "%s/%d/%s.html" % (blog_path, article['date'].year, article['permlink'])
    
def blog_tag_path(tag):
    return "%s/%s.html" % (blog_path, tag)
    
def BlogFinalize():
    global blog_articles, blog_tags
    global blogdb
    global blogdb_articles
    global blogdb_tags

    global blogdb_articles_list
    global blogdb_tags_list
    
    blog_index = {
        'articles': blog_articles,
        'tags': blog_tags
    }
    
    db_hash = sha256(json.dumps(blog_index)).hexdigest();
    f = open("%s/blog.hash" % temporary_directory, "w")
    f.write(db_hash)
    f.close()
    
    for article in blog_articles:
        blogdb[article['permlink']] = article
        article['date'] = blog_date_parse(article['date'])
        if 'update' in article:
            article['update'] = blog_date_parse(article['update'])
        else:
            article['update'] = article['date']
        article['date_display'] = article['date'].astimezone(pytz.timezone(option_display_timezone)).strftime(option_time_format)
        article['update_display'] = article['update'].astimezone(pytz.timezone(option_display_timezone)).strftime(option_time_format)
        article['path'] = blog_article_path(article)

    blogdb_articles = map(lambda article: article['permlink'], blog_index['articles'])
    
    blogdb_articles = blog_sort_articles(blogdb_articles)
    blogdb_articles_list = blog_build_list(blogdb_articles)
    
    # Build chain.
    for i in range(len(blogdb_articles)):
        article = blogdb[blogdb_articles[i]]
        if i > 0:
            article['next'] = copy.deepcopy(blogdb[blogdb_articles[i - 1]])
        if i < len(blogdb_articles) - 1:
            article['prev'] = copy.deepcopy(blogdb[blogdb_articles[i + 1]])
    
    for tag in blogdb_tags_defined:
        blog_tags[tag]['articles'] = blog_sort_articles(blog_tags[tag]['articles'])
        blog_tags[tag]['path'] = blog_tag_path(tag)
        blogdb_tags.append(blog_tags[tag])
        blogdb_tags_map[tag] = blog_tags[tag]
        blogdb_tags_list[tag] = blog_build_list(blog_tags[tag]['articles'])
    
# Generate articles.        
def BlogGenerateArticles(template):
    for permlink in blogdb_articles:
        article = blogdb[permlink]
        opt = article['path']
        Page(opt, article['src'], template)

def BlogGenerateList(target, src, tag, template):
    if tag == "*":
        list = blogdb_articles
    else:
        list = blogdb_tags[tag]['articles']
    def prepare(info):
        return info
    Page(blog_path + "/" + target, src, template, extra = { 'is_list': True, 'list': map(lambda m: prepare(blogdb[m]), list) })

def BlogGenerateTags(src, tag, template):
    def prepare(info):
        return info
    for tag in blogdb_tags:
        list = tag['articles']
        Page(tag['path'], src, template, extra = { 'is_tag': True, 'tag_name': tag['name'], 'list': map(lambda m: prepare(blogdb[m]), list) })    

# Load Website metadata.
execfile("WebsiteMeta")