# Relations

**Relations is a Active Record queries collection based on Rails 4 and Ruby 2**

The magic happens within the migrations, the models and db/seed.rb.

To test the examples just run:

```
$ rake db:setup
```

## Self-referential one-to-one relationship
Everybody has one mother. At most.

```
$ rails g model user first_name:string mother_id:integer
$ rake db:migrate
```

**User model**

```ruby
has_one :mother, class_name: 'User', foreign_key: 'mother_id'
```

**db/seed**

```ruby
john = User.create(first_name: 'John')
heidi = User.create(first_name: 'Heidi')

john.mother = heidi

puts "John's mother is " + Rainbow("#{john.mother.first_name}").yellow
```

```
=> John's mother is Heidi
```

## Join model many-to-many relationship
A user wants to add all videos he likes to a playlist.

```
$ rails g model video title:string engine:string duration:integer
$ rails g model playlist name:string user_id:integer published:boolean
$ rails g model like video_id:integer playlist_id:integer
$ rake db:migrate
```

**Playlist model**

```ruby
belongs_to :user
has_many :likes
has_many :videos, :through => :likes
```

**User model**

```ruby
has_many :playlists
has_many :likes, :through => :playlists
```

**Like model**

```ruby
belongs_to :video
belongs_to :playlist
```

**db/seed**

```ruby
animals = Video.create([
  {title: 'Cat', engine: 'youtube', duration: 90},
  {title: 'Dog', engine: 'youtube', duration: 120}])

# to create a user playlist we use the create method of the association proxy
playlist = john.playlists.create name: 'Animals'
# the foreign keys will be automatically set and the objects will be automatically saved by the << method
playlist.likes << animals.map{|animal| Like.create video: animal}

puts playlist.videos.count
```

```
=> 2
```

```ruby
fruits = Video.create([
  {title: 'Banana', engine: 'vimeo', duration: 140},
  {title: 'Apple', engine: 'dailymotion', duration: 240},
  {title: 'Orange', engine: 'dailymotion', duration: 30}])

playlist = john.playlists.create name: 'Fruits'
# the << method takes both a single associated object or an array of objects
playlist.likes << fruits.map{|fruit| Like.create video: fruit}

puts playlist.videos.count
```

```
=> 3
```

```ruby
john.likes.each{|like| puts like.video.title}
```

```
=> Cat
=> Dog
=> Banana
=> Apple
=> Orange
```

Inspired by [Clipflakes](http://blog.clipflakes.tv/2011/05/26/relaunch-der-website/)' video search engine and playlist creation.


## Order a has_many through association

Now we want a sorted list of all videos John likes. The list has to be sorted by a given video attribute, e.g. the name of the engine or the duration. Instead of iterate through all Like's we want the video list through an additional has_many through association. Example:

```ruby
videos = john.videos.sort(:engine)
```

**User model**

```ruby
has_many :videos, :through => :likes
```

**Video model**

```ruby
scope :sort, ->(column) { order column }
```

Videos sorted by title, engine and duration:

**db/seed**

```ruby
john.videos.sort(:title).each{|video| 
  puts Rainbow("#{video.title}").yellow + " from #{video.engine.titleize} has a duration of #{video.duration} seconds"}

john.videos.sort(:engine).each{|video| 
  puts Rainbow("#{video.engine.titleize}").yellow + "'s #{video.title} has a duration of #{video.duration} seconds"}

john.videos.sort(:duration).each{|video| 
  puts Rainbow("#{video.duration}").yellow + " seconds is the duration of #{video.title} from #{video.engine.titleize}"}
```

```
=> Apple from Dailymotion has a duration of 240 seconds
=> Banana from Vimeo has a duration of 140 seconds
=> Cat from Youtube has a duration of 90 seconds
=> Dog from Youtube has a duration of 120 seconds
=> Orange from Dailymotion has a duration of 30 seconds

=> Dailymotion Apple has a duration of 240 seconds
=> Dailymotion Orange has a duration of 30 seconds
=> Vimeo Banana has a duration of 140 seconds
=> Youtube Cat has a duration of 90 seconds
=> Youtube Dog has a duration of 120 seconds

=> 30 seconds is the duration of Orange from Dailymotion
=> 90 seconds is the duration of Cat from Youtube
=> 120 seconds is the duration of Dog from Youtube
=> 140 seconds is the duration of Banana from Vimeo
=> 240 seconds is the duration of Apple from Dailymotion
```

The ActiveRecord::Base order method works the same. It's one of the association collection wrapper methods.

```ruby
john.videos.order(:title).each{|video| 
  puts Rainbow("#{video.title}").yellow + " from #{video.engine.titleize} has a duration of #{video.duration} seconds"}
```

```
=> Apple from Dailymotion has a duration of 240 seconds
=> Banana from Vimeo has a duration of 140 seconds
=> Cat from Youtube has a duration of 90 seconds
=> Dog from Youtube has a duration of 120 seconds
=> Orange from Dailymotion has a duration of 30 seconds
```

Inspired by the article [Sorting and Reordering Has Many Through Associations With the ActsAsList Gem](http://easyactiverecord.com/blog/2014/11/11/sorting-and-reordering-lists-with-the-actsaslist-gem/).


## Combined scope conditions

While the sort scope in the previous example was just another way to order a query result we now want a new scope condition for returning only videos with a given minimum length.

**Video model**

```ruby
scope :duration_min, ->(seconds) { where(duration: seconds.to_i..Float::INFINITY) }
```

Scopes can be combined: display all videos with at least 100 seconds ordered by their engine names:

**db/seed**

```ruby
john.videos.duration_min(100).sort(:engine).each{|video| 
  puts Rainbow("#{video.engine.titleize}").yellow + " #{video.title} has a duration of #{video.duration} seconds"}
```

```
=> Dailymotion Apple has a duration of 240 seconds
=> Vimeo Banana has a duration of 140 seconds
=> Youtube Dog has a duration of 120 seconds
```

## Merging scopes with joins

We now want all videos from a specific playlist. We can do this by combining a join with a merge condition.

**Video model**

```ruby
has_many :likes
has_many :playlist, :through => :likes
scope :list, ->(name) { joins(:playlist).merge( Playlist.where(name: name) )}
```

**db/seed**

```ruby
%w[Animals Fruits].each do |name|
  # use of association collection average method
  puts "Playlist " + Rainbow("#{name}").yellow + " has an average duration of " + Rainbow("#{john.videos.list(name).average(:duration).to_i}").yellow + " seconds"
  john.videos.list(name).each{|video| 
  puts "Playlist " + Rainbow("#{name}").yellow + " has video " + Rainbow("#{video.title}").yellow + " from #{video.engine.titleize}"}
end
```

```
=> Playlist Animals has an average duration of 105 seconds
=> Playlist Animals has video Cat from Youtube
=> Playlist Animals has video Dog from Youtube
=> Playlist Fruits has an average duration of 136 seconds
=> Playlist Fruits has video Banana from Vimeo
=> Playlist Fruits has video Apple from Dailymotion
=> Playlist Fruits has video Orange from Dailymotion
```

*Merge can also call a scope*

Let's search for all playlists that have at least one Vimeo video.

**Video model**

```ruby
scope :vimeo, -> { where(engine: 'vimeo') }
```

**Playlist model**

```ruby
scope :vimeo, -> { joins(:videos).merge( Video.vimeo ) }
```

**db/seed**

```ruby
# how many videos from Vimeo do we have?
puts Video.vimeo.count
# display the title of the first Vimeo video in the first playlist that has at least one video from Vimeo
puts john.playlists.vimeo.first.videos.vimeo.first.title
```

```
#=> 1
#=> Banana
```

Inspired by [Railscasts '#215 Advanced Queries in Rails 3'](http://railscasts.com/episodes/215-advanced-queries-in-rails-3)
