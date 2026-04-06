require 'cgi'
require 'commonmarker'
require 'fspath'
require 'json'
require 'net/http'
require 'nokogiri'
require 'yaml/store'

docset = 'GitHub_Actions.docset'

PATH_R = %r{^/en/actions(?:/.+|$)}

task default: "#{docset}.tgz"

file "#{docset}.tgz" => "#{docset}/.done" do |t|
  sh *%W[
    docker run --rm -i -v #{Dir.pwd}:/here -w /here ubuntu
    tar --exclude=.{DS_Store,done} -cvzf #{t.name} #{File.dirname(t.source)}
  ]
end

file "#{docset}/.done" => 'html/.done' do |t|
  dir = FSPath(t.name).dirname
  rm_rf dir
  mkdir_p dir

  sh "dashing build --source #{File.dirname(t.source)}"

  touch t.name
end

file 'html/.done' => ['data/.done', 'Rakefile'] do |t|
  dir = FSPath(t.name).dirname
  rm_rf dir
  mkdir_p dir

  ln 'icon.png', dir
  ln 'GitHub documentation LICENSE.txt', dir

  Transformer.new(File.dirname(t.source), File.dirname(t.name)).write

  touch t.name
end

file 'data/.done' do |t|
  dir = FSPath(t.name).dirname
  rm_rf dir
  mkdir_p dir

  pages = Fetch.get('https://docs.github.com/api/pagelist/en/free-pro-team@latest').scan(PATH_R)

  unexpected = pages.grep_v(%r{\A(/[a-z]+([\-_][a-z]+)*)+\z})
  abort "unexpected paths #{unexpected}" unless unexpected.empty?

  pages.each do |page|
    (dir / "#{page}.json").carefull_write{ Fetch.get("https://docs.github.com/api/article?pathname=#{page}") }
  end

  touch t.name
end

task :clean do
  rm_rf %W[#{docset}.tgz #{docset} html data]
end

FSPath.class_eval do
  def carefull_write(content = nil)
    return if size? && (content.nil? || read == content)

    content ||= yield

    temp_file_path(dirname.mkpath) do |tmp|
      tmp.write(content)
      tmp.rename(self)
    end
  end
end

module Fetch
  PERIOD = 1
  MAX_REQUESTS = 5

  @times = []

module_function

  def get_response(url)
    wait?
    puts url
    Net::HTTP.get_response(URI(url))
  end

  def wait?
    now = nil
    loop do
      now = Process.clock_gettime(Process::CLOCK_MONOTONIC)
      @times.reject!{ |t| now - t > PERIOD }
      break if @times.size < MAX_REQUESTS

      to_sleep = PERIOD - (now - @times.first)
      sleep(to_sleep) if to_sleep > 0
    end
    @times << now
  end

  def get(url)
    response = get_response(url)
    abort "got #{response.class} with #{response.body}" unless response.is_a?(Net::HTTPSuccess)

    response.body
  end

  def get_redirect(url)
    response = get_response(url)
    abort "got #{response.class} with #{response.body}" unless response.is_a?(Net::HTTPMovedPermanently)

    response['Location']
  end
end

class Transformer
  Json = Data.define(:path, :data) do
    def title = meta['title']
    def body = data['body']
    def meta = data['meta']
    def folder? = meta['documentType'] != 'article'
    def breadcrumbs = meta['breadcrumbs']
  end

  def initialize(data_dir, html_dir)
    @data_dir = FSPath(data_dir)
    @html_dir = FSPath(html_dir)
    @jsons = Dir.chdir(data_dir) do
      FSPath.glob('**/*.json').map do |path|
        Json.new(path.to_s.delete_suffix('.json'), ::JSON.parse(path.read))
      end.sort_by(&:path)
    end
    @known_paths = @jsons.each_with_object({}) do |json, paths|
      paths["/#{json.path}"] = "#{json.path}#{'/index' if json.folder?}.html"
    end
    @redirects = YAML::Store.new("#{data_dir}/redirects.yml")
    @css_path = 'assets/style.css'
  end

  def write
    write_css

    @redirects.transaction do
      @jsons.each do |json|
        write_html(json)
      end
    end
  end

private

  def write_css
    dst = @html_dir / @css_path

    css = {
      light: 'https://raw.githubusercontent.com/sindresorhus/github-markdown-css/gh-pages/github-markdown-light.css',
      dark: 'https://raw.githubusercontent.com/sindresorhus/github-markdown-css/gh-pages/github-markdown-dark-dimmed.css',
    }.to_h do |name, url|
      cache = @data_dir / "#{name}.css"
      cache.carefull_write{ Fetch.get(url) }
      [name, cache.read]
    end

    dst.dirname.mkpath
    dst.write <<~CSS
      body { padding: 40px; }

      a.external::after {
        content: "↗";
        padding-left: 0.25em;
      }

      #{css[:light]}
      @media (prefers-color-scheme: dark) {
        #{css[:dark]}
      }
    CSS
  end

  def write_html(json)
    html_path = @known_paths["/#{json.path}"]

    (@html_dir / html_path).carefull_write(markdown_to_html(json, html_path))
  end

  def markdown_to_html(json, html_path)
    fixed_markdown = json.body.gsub(/^( *> )\\(\[!(?:NOTE|TIP|IMPORTANT|WARNING|CAUTION)\])(.*)/) do
      $3 == '' ? "#{$1}#{$2}" : "#{$1}#{$2}\n#{$1}#{$3}"
    end

    html = Commonmarker.to_html(
      fixed_markdown,
      options: {
        render: {
          unsafe: true,
        },
        extension: {
          alerts: true,
        },
      },
    )

    fragment = Nokogiri::HTML::DocumentFragment.parse(html)

    fragment.search('a[href]').each do |a|
      href = a['href']
      unless %r{\A(?<path>/[^#]*)(?<anchor>#.*)?} =~ href
        a['class'] = [a['class'], 'external'].compact.join(' ') unless href.start_with?('#')
        next
      end

      if path =~ PATH_R && !@known_paths[path]
        href = (@redirects[path] ||= Fetch.get_redirect("https://docs.github.com#{href}"))

        abort "unexpected redirect #{path} to #{href}" unless %r{\A(?<path>/[^#]*)(?<redirect_anchor>#.*)?} =~ href

        if redirect_anchor
          warn "got anchor #{redirect_anchor} in redirect (#{path} => #{href})"
          href = path
        end

        path, query = path.split('?', 2)
        if query
          warn "removing query from link #{href}"
          href = path
        end
      end

      if path =~ PATH_R
        abort "unknown path #{path}" unless link_path = @known_paths[path]

        a['href'] = "#{relative_path(link_path, html_path)}#{anchor}"
      else
        a['href'] = "https://docs.github.com#{href}"
        a['class'] = [a['class'], 'external'].compact.join(' ')
      end
    end

    fragment.search('img[src]').each do |img|
      src = img['src']
      abort "unexpected path #{src}" unless src =~ %r{\A/assets(/[a-z0-9]+([\-_][a-z0-9]+)*)+\.png\z}

      cache = @data_dir / src
      cache.carefull_write{ Fetch.get("https://docs.github.com#{src}") }

      dst = @html_dir / src
      unless dst.size?
        dst.dirname.mkpath
        dst.make_link(cache)
      end

      img['src'] = relative_path(src, html_path)
    end

    if json.breadcrumbs.length > 1
      nav = Nokogiri::XML::Node.new('nav', fragment)

      json.breadcrumbs[...-1].each do |breadcrumb|
        path = breadcrumb['href']
        title = breadcrumb['title']
        abort "unknown path #{path}" unless link_path = @known_paths[path]

        a = Nokogiri::XML::Node.new('a', fragment)
        a['href'] = relative_path(link_path, html_path)
        a.content = title

        nav.add_child(a)
        nav.add_child(' / ')
      end

      fragment.children.first.before(nav)
    end

    index = []
    fragment.search('h2, h3').each do |h|
      index[h.name[1].to_i - 2..] = h.text

      h['title'] = if index[0] && index[1] && index[1].include?(index[0])
        index[1]
      else
        index.compact.join(' / ')
      end
    end

    # try to find meaningfull keywords in tables
    # find only columns which contain only single <code> and all texts should be different
    # there should be only one such column
    fragment.search('tbody').each do |tbody|
      rows = tbody.search('tr').map do |tr|
        tr.search('td').map do |td|
          codes = td.search('code')

          codes.first if codes.length == 1 && codes.text == td.text
        end
      end

      columns = rows.transpose.reject{ |c| c.length < 2 || c.include?(nil) || c.uniq(&:text).length < c.length }
      next unless columns.length == 1

      columns.first.each{ |code| code['class'] = 'keyword' }
    end

    <<~HTML
      <!DOCTYPE html>
      <html>
        <head>
          <meta charset="utf-8">
          <title>#{h json.title}</title>
          <link rel="stylesheet" href="#{h relative_path(@css_path, html_path)}">
        </head>
        <body class="markdown-body">
          #{fragment}

          <footer>
            Documentation is licensed under Creative Commons Attribution 4.0
            (see <a href="#{h relative_path('GitHub documentation LICENSE.txt', html_path)}">GitHub documentation LICENSE</a>)
          </footer>
        </body>
      </html>
    HTML
  end

  def relative_path(to, from)
    return '' if to == from

    to_parts = to.split('/')
    from_parts = from.split('/')
    same = to_parts.zip(from_parts).take_while{ _1 == _2 }.length
    "#{'../' * (from_parts.length - same - 1)}#{to_parts.drop(same).join('/')}"
  end

  def h(s) = CGI.escapeHTML(s)
end
