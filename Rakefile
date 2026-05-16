exec 'rake' if __FILE__ == $0

require 'commonmarker'
require 'date'
require 'fspath'
require 'json'
require 'net/http'
require 'nokogiri'
require 'yaml/store'

FSPath.class_eval do
  def done = self / '.done'

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
  class Unexpected < StandardError; end

  PERIOD = 1
  MAX_REQUESTS = 5

  @times = []

  class << self
    def get(url)
      response = get_response(url)
      return response.body if response.is_a?(Net::HTTPSuccess)

      fail Unexpected, "instead of #{Net::HTTPSuccess} got #{response.class} with #{response.body}"
    end

    def get_redirect(url)
      response = get_response(url)
      return response['Location'] if response.is_a?(Net::HTTPMovedPermanently)

      fail Unexpected, "instead of #{Net::HTTPMovedPermanently} got #{response.class}"
    end

  private

    def get_response(url)
      wait
      puts "fetching #{url}"
      Net::HTTP.get_response(URI(url))
    end

    def wait
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
  end
end

class GithubDocBuilder
  Json = Data.define(:path, :data) do
    def title = meta['title']
    def body = data['body']
    def meta = data['meta']
    def folder? = meta['documentType'] != 'article'
    def breadcrumbs = meta['breadcrumbs']
  end

  include Rake::DSL
  extend Rake::DSL

  CLEAN_PATHS = [
    CACHE_MARK_PATH = '.cache-mark',
    PAGELIST_PATH = '.pagelist',
    CSS_PATH = '.css',
    DATA_DIR = 'data',
    HTML_DIR = 'html',
    DASHING_CONFIGS_DIR = 'dashing-configs',
    DOCSETS_DIR = 'docsets',
    ARCHIVES_DIR = 'archives',
    RESULTS_DIR = 'results',
  ]

  CSS_URLS = {
    light: 'https://raw.githubusercontent.com/sindresorhus/github-markdown-css/gh-pages/github-markdown-light.css',
    dark: 'https://raw.githubusercontent.com/sindresorhus/github-markdown-css/gh-pages/github-markdown-dark-dimmed.css',
  }

  def self.all
    yield new 'actions'
    yield new 'code-security', name: 'GitHub Security and code quality'
    yield new 'pages'
  end

  def self.define_tasks
    if File.exist?(CACHE_MARK_PATH) && Time.now - File.mtime(CACHE_MARK_PATH) > 24 * 60 * 60
      File.unlink(CACHE_MARK_PATH)
    end

    file CACHE_MARK_PATH do |t|
      touch t.name
    end

    file PAGELIST_PATH => CACHE_MARK_PATH do |t|
      FSPath(t.name).carefull_write do
        Fetch.get('https://docs.github.com/api/pagelist/en/free-pro-team@latest')
      end
    end

    file CSS_PATH => CACHE_MARK_PATH do |t|
      css = CSS_URLS.transform_values{ |url| Fetch.get(url) }

      FSPath(t.name).carefull_write <<~CSS
        body { padding: 40px; }

        a.external::after {
          content: "↗";
          padding-left: 0.25em;
        }

        #{css[:light]}
        @media (prefers-color-scheme: dark) {
          #{css[:dark].gsub(/^/, '  ')}
        }
      CSS
    end

    all(&:define_tasks)

    task :clean do
      rm_rf CLEAN_PATHS
    end
  end

  attr_reader :prefix, :name
  attr_reader :version
  attr_reader :package, :path_r
  attr_reader :data_path, :html_path, :dashing_config_path, :docset_path, :archive_path, :result_path

  def initialize(prefix, name: "GitHub #{prefix.capitalize}")
    @prefix = prefix
    @name = name

    @version = Date.today.strftime('%Y.%m')

    @package = name.tr(' ', '_')
    @path_r = %r{^/en/#{prefix}(?:/.+|$)}

    @data_path = FSPath(DATA_DIR) / package
    @html_path = FSPath(HTML_DIR) / package
    @dashing_config_path = FSPath(DASHING_CONFIGS_DIR) / "#{package}-#{version}.json"
    @docset_path = FSPath(DOCSETS_DIR) / "#{package}.docset"
    @archive_path = FSPath(ARCHIVES_DIR) / "#{package}.docset.tgz"
    @result_path = FSPath(RESULTS_DIR) / package
  end

  def define_tasks
    dir data_path => PAGELIST_PATH do
      pages = File.read(PAGELIST_PATH).scan(path_r)

      unexpected = pages.grep_v(%r{\A(/[a-z0-9]+([\-_][a-z0-9]+)*)+\z})
      abort "unexpected paths #{unexpected}" unless unexpected.empty?

      pages.each do |page|
        (data_path / "#{page}.json").carefull_write do
          Fetch.get("https://docs.github.com/api/article?pathname=#{page}")
        end
      end
    end

    dir html_path => [CSS_PATH, data_path.done] do
      mkdir_p html_path

      ln 'GitHub documentation LICENSE.txt', html_path

      generate_html
    end

    file dashing_config_path => html_path.done do
      dashing_config_path.carefull_write(dashing_config)
    end

    dir docset_path => [dashing_config_path, html_path.done] do
      mkdir_p docset_path.dirname

      sh(*%W[dashing build --source #{html_path} --config #{dashing_config_path}])

      mv docset_path.basename, docset_path.dirname

      %w[icon.png icon@2x.png].each do |icon_basename|
        ln icon_basename, docset_path
      end
    end

    file archive_path => docset_path.done do
      mkdir_p archive_path.dirname

      sh(*docker_run_args('debian') + %W[
        tar
        --directory=#{docset_path.dirname}
        --exclude=.DS_Store
        -cvzf
        #{archive_path}
        #{docset_path.basename}
      ])
    end

    docset_meta_path = result_path / 'docset.json'

    results = {
      result_path / archive_path.basename => archive_path,
      **%w[README.md icon.png icon@2x.png].to_h do |basename|
        [result_path / basename, basename]
      end,
    }

    file docset_meta_path => results.values do
      rm_rf result_path
      mkdir_p result_path

      results.each do |dst, src|
        ln src, dst
      end

      docset_meta_path.carefull_write(docset_meta)
    end

    task default: docset_meta_path
  end

private

  def dir(arg)
    fail ArgumentError, 'expected Hash with 1 pair' unless arg.is_a?(Hash) && arg.length == 1

    dir, dependencies = arg.first
    done = dir.done

    file done => dependencies do
      rm_rf dir

      yield

      touch done
    end
  end

  def dashing_config
    JSON.pretty_generate(
      name:,
      package:,
      index: "#{html_path}/en/#{prefix}/index.html",
      externalURL: "https://docs.github.com/en/#{prefix}",
      selectors: {
        'h1' => 'Guide',
        'h2, h3' => {attr: 'title', type: 'Section'},
        'code.keyword' => 'Keyword',
      },
    )
  end

  def docset_meta
    JSON.pretty_generate({
      name:,
      version:,
      archive: archive_path.basename,
      author: {
        name: 'Ivan Kuchin',
        link: 'https://github.com/toy',
      },
    })
  end

  def generate_html
    jsons = chdir(data_path) do
      FSPath.glob('**/*.json').map do |path|
        Json.new(path.to_s.delete_suffix('.json'), ::JSON.parse(path.read))
      end.sort_by(&:path)
    end

    known_paths = jsons.each_with_object({}) do |json, paths|
      paths["/#{json.path}"] = "#{json.path}#{'/index' if json.folder?}.html"
    end

    redirects = YAML::Store.new(data_path / 'redirects.yml')

    relative_css_path = 'assets/style.css'
    css_path = html_path / relative_css_path

    mkdir_p css_path.dirname
    ln CSS_PATH, html_path / relative_css_path

    jsons.each do |json|
      redirects.transaction do
        relative_file_path = known_paths["/#{json.path}"]

        html = markdown_to_html(json:, relative_file_path:, known_paths:, relative_css_path:, redirects:)

        (html_path / relative_file_path).carefull_write(html)
      end
    end
  end

  def markdown_to_html(json:, relative_file_path:, known_paths:, relative_css_path:, redirects:)
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

    fragment = Nokogiri::HTML5::DocumentFragment.parse(html)

    fragment.search('a[href]').each do |a|
      href = a['href']
      unless %r{\A(?<path>/[^#]*)(?<anchor>#.*)?} =~ href
        a['class'] = [a['class'], 'external'].compact.join(' ') unless href.start_with?('#')
        next
      end

      if path =~ path_r && !known_paths[path]
        href = (redirects[path] ||= Fetch.get_redirect("https://docs.github.com#{href}"))

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

      if path =~ path_r
        abort "unknown path #{path}" unless link_path = known_paths[path]

        a['href'] = "#{relative_path(link_path, relative_file_path)}#{anchor}"
      else
        a['href'] = "https://docs.github.com#{href}"
        a['class'] = [a['class'], 'external'].compact.join(' ')
      end
    rescue Fetch::Unexpected
      warn "ignoring #{path}"
    end

    fragment.search('img[src]').each do |img|
      src = img['src']
      abort "unexpected path #{src}" unless src =~ %r{\A/assets(/[a-z0-9]+([\-_+][a-z0-9]+)*)+\.png\z}

      cache = data_path / src
      cache.carefull_write{ Fetch.get("https://docs.github.com#{src}") }

      dst = html_path / src
      unless dst.size?
        dst.dirname.mkpath
        dst.make_link(cache)
      end

      img['src'] = relative_path(src, relative_file_path)
    end

    if json.breadcrumbs.length > 1
      nav = Nokogiri::XML::Node.new('nav', fragment)

      json.breadcrumbs[...-1].each do |breadcrumb|
        path = breadcrumb['href']
        title = breadcrumb['title']
        abort "unknown path #{path}" unless link_path = known_paths[path]

        a = Nokogiri::XML::Node.new('a', fragment)
        a['href'] = relative_path(link_path, relative_file_path)
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

    Nokogiri::HTML5::Builder.new do |doc|
      doc.html do
        doc.head do
          doc.meta charset: 'utf-8'
          doc.title json.title
          doc.link rel: 'stylesheet', href: relative_path(relative_css_path, relative_file_path)
        end
        doc.body class: 'markdown-body' do
          doc << fragment
          doc.footer do
            doc.text 'Documentation is licensed under Creative Commons Attribution 4.0 (see '
            doc.a 'GitHub documentation LICENSE', href: relative_path('GitHub documentation LICENSE.txt', relative_file_path)
            doc.text ')'
          end
        end
      end
    end.to_html
  end

  def relative_path(to, from)
    return '' if to == from

    to_parts = to.split('/')
    from_parts = from.split('/')
    same = to_parts.zip(from_parts).take_while{ _1 == _2 }.length
    "#{'../' * (from_parts.length - same - 1)}#{to_parts.drop(same).join('/')}"
  end

  def docker_run_args(image)
    %W[
      docker run
      --user #{Process.uid}:#{Process.gid}
      --rm
      -i
      -v #{Dir.pwd}:/here
      -w /here
      #{image}
    ]
  end
end

GithubDocBuilder.define_tasks
