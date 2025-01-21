require 'erb'
require 'json'
require 'open-uri'
require 'rss'
require 'timezone'

STORE = '028' # IKEA Portland
TIMEZONE = Timezone['America/Los_Angeles']
MAX_PAGE_SIZE = 64
LINK_TEMPLATE = 'https://www.ikea.com/us/en/customer-service/shopping-at-ikea/as-is-online-pubce1eedc0/#/portland/%s'

class IkeaDiff
  attr_reader :previous_results, :current_results

  def initialize(previous_results, current_results)
    @previous_results = previous_results&.to_h { |r| [r['id'], r] } || {}
    @current_results = current_results&.to_h { |r| [r['id'], r] } || {}
  end

  def current_ids
    current_results.keys
  end

  def previous_ids
    previous_results.keys
  end

  def added_ids
    current_ids - previous_ids
  end

  def deleted_ids
    previous_ids - current_ids
  end

  def added_items
    current_results.values_at(*added_ids)
  end

  def deleted_items
    previous_results.values_at(*deleted_ids)
  end

  def grouped_added_items
    @grouped_added_items ||= added_items.group_by { |item| [item['title'], item['description'], item['price']].join('/') }
  end

  def grouped_deleted_items
    @grouped_deleted_items ||= deleted_items.group_by { |item| [item['title'], item['description'], item['price']].join('/') }
  end

  def edited_items
    @edited_items ||= current_ids.filter_map do |id|
      current_result = current_results[id]
      previous_result = previous_results[id]
      next if current_result == previous_result || current_result.nil? || previous_result.nil?

      current_result.merge(
        'changes' => current_result.keys.filter_map do |key|
          next if current_result[key] == previous_result[key]

          [
            key,
            [previous_result[key], current_result[key]]
          ]
        end.to_h
      )
    end
  end

  def changed_items
    added_items.map { |item| item.merge('change_type' => 'added') } +
      deleted_items.map { |item| item.merge('change_type' => 'deleted') } +
      edited_items.map { |item| item.merge('change_type' => 'edited') }
  end

  def grouped_changed_items
    @grouped_changed_items ||= changed_items.group_by { |item| [item['change_type'], item['title'], item['description'], item['price']].join('/') }
  end
end

directory 'build'
directory 'data'

task update: 'data' do
  results = []
  page = 0
  total_pages = 1

  while page < total_pages
    uri = URI("https://web-api.ikea.com/circular/circular-asis/offers/public/us/en?size=#{MAX_PAGE_SIZE}&stores=#{STORE}&column=id&direction=desc&page=#{page}")
    response = JSON.parse(uri.open.read)
    total_pages = response['totalPages'].to_i
    results += response['content']
    page += 1
  end

  previous_file = Dir.glob('data/*.json').last
  previous_results = previous_file && JSON.parse(File.read(previous_file))['results'] || []
  next if previous_results == results

  File.write("data/#{Time.now.iso8601}.json", JSON.pretty_generate(results: results))
end

TXT_TEMPLATE = <<~TEMPLATE.chomp
  <% revisions.each do |revision| %>
  ===[ <%= revision[:time] %> ]=====================================

  <% diff = revision[:diff] %>
  <% if diff.added_items.any? %>
  Added items:
  <% diff.grouped_added_items.each do |group, items| %>
  + <%= items.count %>X <%= items[0]['title'] %> (<%= '$%.2f' % items[0]['price'] %>): <%= items[0]['description'] %>
  <% end %>
  <% end %>

  <% if diff.deleted_items.any? %>
  Deleted items:
  <% diff.grouped_deleted_items.each do |group, items| %>
  + <%= items.count %>X <%= items[0]['title'] %> (<%= '$%.2f' % items[0]['price'] %>): <%= items[0]['description'] %>
  <% end %>
  <% end %>

  <% if diff.edited_items.any? %>
  Edited items:
  <% diff.edited_items.each do |item| %>
  * <%= item['title'] %> (<%= '$%.2f' % item['price'] %>): <%= item['description'] %>
  <% item['changes'].each do |key, (previous, current)| %>
  - <%= key %>: <%= previous %> -> <%= current %>
  <% end %>
  <% end %>
  <% end %>
  <% end %>
TEMPLATE

file 'build/recent_changes.txt' => %w[build update] do |t|
  revisions = Dir.glob('data/*.json').last(10).reverse.each_cons(2).map do |current_file, previous_file|
    current = JSON.parse(File.read(current_file))['results']
    previous = JSON.parse(File.read(previous_file))['results']
    diff = IkeaDiff.new(previous, current)

    { time: Time.parse(current_file.pathmap('%n')).getlocal(TIMEZONE), diff: diff }
  end

  File.write(t.name, ERB.new(TXT_TEMPLATE, trim_mode: "<>").result(binding))
end

HTML_TEMPLATE = <<~TEMPLATE.chomp
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>IKEA As-Is Changes</title>
  <style>
    body { font-family: Arial, sans-serif; margin: 20px; }
    h2 { border-bottom: 1px solid #ccc; padding-bottom: 5px; }
    h4 { margin-bottom: 0.5em; }
    p { margin-top: 0; }
    .items {
      display: grid;
      grid-template-columns: repeat(auto-fill, 220px);
      grid-column-gap: 10px;
      grid-row-gap: 10px;
    }
    .item {
      color: black;
      text-decoration: none;
      display: block;
      border: 1px solid #ccc;
      padding: 10px;
    }
    .item:hover { border: 1px dashed blue; }
    .item.deleted { border-color: red; opacity: 0.5; }
    .edited { color: orange; }
    .change { margin-left: 20px; }
    .discount-reason {
      color: #666;
      font-size: 0.75em;
      font-weight: bold;
      text-transform: uppercase;
    }
    .original-price { text-decoration: line-through; }
    img { max-width: 200px; display: block; margin-top: 10px; }
  </style>
</head>
<body>
  <% revisions.each do |revision| %>
    <h2><%= revision[:time].strftime('%m/%d/%Y %I:%M%p') %></h2>

    <% diff = revision[:diff] %>
    <div class="items">
      <% diff.grouped_changed_items.each do |group, items| %>
        <% item = items[0] %>
        <a href="<%= LINK_TEMPLATE % item['id'] %>" class='<%= item['change_type'] %> item'>
            <img src="<%= item['heroImage'] %>" alt="<%= item['title'] %>">
          <h4 class='title'>
            <%= item['title'] %>
            <% if items.count > 1 %>
              <span class='count'>x <%= items.count %></span>
            <% end %>
          </h4>
          <p class='description'><%= item['description'] %></p>
          <p class='discount-reason'><%= item['reasonDiscount'] %></p>
          <p class='price'>
            <%= '$%.2f' % item['price'] %>
            <span class='original-price'><%= '$%.2f' % item['articlesPrice'] %></span>
          </p>
        </a>
      <% end %>
    </div>
  <% end %>
</body>
</html>
TEMPLATE

file 'build/index.html' => %w[build update] do |t|
  revisions = Dir.glob('data/*.json').last(10).reverse.each_cons(2).map do |current_file, previous_file|
    current = JSON.parse(File.read(current_file))['results']
    previous = JSON.parse(File.read(previous_file))['results']
    diff = IkeaDiff.new(previous, current)

    { time: Time.parse(current_file.pathmap('%n')).getlocal(TIMEZONE), diff: diff }
  end

  File.write(t.name, ERB.new(HTML_TEMPLATE, trim_mode: "<>").result(binding))
end

file 'build/changes.atom' => %w[build update] do |t|
  revisions = Dir.glob('data/*.json').last(10).reverse.each_cons(2).map do |current_file, previous_file|
    current = JSON.parse(File.read(current_file))['results']
    previous = JSON.parse(File.read(previous_file))['results']
    diff = IkeaDiff.new(previous, current)

    { time: Time.parse(current_file.pathmap('%n')).getlocal(TIMEZONE), diff: diff }
  end

  rss = RSS::Maker.make("atom") do |maker|
    maker.channel.author = "IKEA As-Is"
    maker.channel.updated = Time.now.to_s
    maker.channel.about = "IKEA As-Is Changes"
    maker.channel.title = "IKEA As-Is Changes"

    revisions.each do |revision|
      revision[:diff].changed_items.each do |item|
        maker.items.new_item do |entry|
          entry.link = LINK_TEMPLATE % item['id']
          entry.title = "#{item['change_type'].capitalize}: #{item['title']}"
          entry.summary = <<~SUMMARY
            <img src='#{item['heroImage']}' alt='#{item['title']}' /><br/>
            #{item['description']}<br/>
            <strong>Discount Reason:</strong> #{item['reasonDiscount']}<br/>
            <strong>Price:</strong> #{'$%.2f' % item['price']}<br/>
            <strong>Original Price:</strong> <span style='text-decoration: line-through;'>#{'$%.2f' % item['articlesPrice']}</span>
          SUMMARY
          entry.updated = revision[:time]
        end
      end
    end
  end

  File.write(t.name, rss.to_s)
end

file 'build/items.atom' => %w[build update] do |t|
  latest_file = Dir.glob('data/*.json').last
  latest_results = JSON.parse(File.read(latest_file))['results']

  item_first_seen = {}

  Dir.glob('data/*.json').last(10).reverse.each do |file|
    results = JSON.parse(File.read(file))['results']
    results.each do |item|
      item_first_seen[item['id']] ||= Time.parse(file.pathmap('%n')).getlocal(TIMEZONE).to_s
    end
  end

  rss = RSS::Maker.make("atom") do |maker|
    maker.channel.author = "IKEA As-Is"
    maker.channel.updated = Time.now.to_s
    maker.channel.about = "IKEA As-Is Items"
    maker.channel.title = "IKEA As-Is Items"

    latest_results.each do |item|
      maker.items.new_item do |entry|
      entry.link = LINK_TEMPLATE % item['id']
      entry.title = item['title']
      entry.summary = <<~SUMMARY
        <img src='#{item['heroImage']}' alt='#{item['title']}' /><br/>
        #{item['description']}<br/>
        <strong>Discount Reason:</strong> #{item['reasonDiscount']}<br/>
        <strong>Price:</strong> #{'$%.2f' % item['price']}<br/>
        <strong>Original Price:</strong> <span style='text-decoration: line-through;'>#{'$%.2f' % item['articlesPrice']}</span>
      SUMMARY
      entry.updated = item_first_seen[item['id']]
      end
    end
  end

  File.write(t.name, rss.to_s)
end

task :default => %w(build/index.html build/items.atom build/changes.atom)



