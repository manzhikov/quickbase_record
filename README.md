# QuickbaseRecord

QuickbaseRecord is a baller ActiveRecord-style ORM for using the Intuit QuickBase platform as a database for Ruby or Ruby on Rails applications.

## Installation

Add this line to your application's Gemfile:

```ruby
gem 'quickbase_record'
```

And then execute:

    $ bundle

Or install it yourself as:

    $ gem install quickbase_record

## Usage

### Initialize the API Client
QuickbaseRecord is built on top of the [Advantage Quickbase](https://github.com/AdvantageIntegratedSolutions/Quickbase-Gem) gem to make the API calls to QuickBase, so you'll need to configure QuickbaseRecord with your app's realm name and provide a valid username, password, and token (if applicable). This can be done in a single initializer file with a call to `QuickbaseRecords.configure`.

```
  # config/initializers/quickbase_record.rb

  QuickbaseRecord.configure do |config|
    config.realm = "quickbase_app_realm_name"
    config.username = "valid_username"
    config.password = "valid_password"
    config.token = "quickbase_app_token_if_applicable"
  end
```

### Include it in your Class
Simply `include QuickbaseRecord::Model` in your class and use the `.define_fields` method to supply the table's DBID and a mapping of desired field names => QuickBase FIDs.

**(NEW IN 0.4.0)**
.define_fields follows a similar pattern to ActiveRecord migrations. It takes a block where data types and field names are definied with a corresponding QuickBase FID.

```
  # app/models/post.rb

  class Post
    include QuickbaseRecord::Model

    define_fields do |t|
      t.dbid 'abcde12345'
      t.number :id, 3, :primary_key
      t.string :content, 7
      t.string :author, 8
      t.string :title, 9
      t.string :title_plus_author, 10, :read_only
    })

    # code...
  end
```

### .define_fields
.define_fields(:field_name, fid, *options)

The following data types are currently supported:
- "dbid"
  - The dbid for the QuickBase table
  - **Does not take multiple arguments, only a string of the dbid**
- "string"
  - All values are converted to a String
- "number"
  - All values are converted to a Numeric (and knows if it needs to be a float)
- "date"
  - All values are converted to a string representation of the date value: "07/30/2015" or nil for empty values
- "file_attachment"
  - Doesn't really do anything, only makes you feel better :) File attachments are explained below.

Additional options may be added to field definitions:
- :primary_key
  - This is a required option for one field.
- :read_only
  - Fields marked as :read_only will not respond to #save or #update_attributes calls.
  - Useful for formula/lookup/other fields in your QuickBase table that you can't write to.


**IMPORTANT: You must supply a "dbid" data type and mark a single field as :primary_key**

### Queries
To query for records you can use the .find, .where, or .qid methods on the class. See below for examples.

If you want to see the QuickBase query string output of a .where() argument you can pass your query hash to the .build_query() method and it will return the QuickBase query.

```
Post.build_query(author: 'Cullen', title: 'Some Title')
=> "{'8'.EX.'Cullen'}AND{'9'.EX.'Some Title'}"
```

### What You Get
Classes that include QuickbaseRecord::Model and define their fields will have a handful of class and instance methods for interacting with your QuickBase application similar to ActiveRecord. The goal is to be able to use QuickBase as a database and treat your models the same way you would with a traditional database.

```
@post = Post.find(1) => <Post: @id=1, @content="Amazing post content", @author: 'Cullen Jett'>

--

<%= form_for @post do |f| %>
  # code...
<% end %>

--

@post = Post.where(author: 'Cullen Jett').first
@post.update_attributes(author: 'THE Cullen Jett')

--

etc.
```


Also included/extended are ActiveModel::Naming, ActiveModel::Callbacks, ActiveModel::Validations, and ActiveModel::Conversion ([see ActiveModel docs for details](https://github.com/rails/rails/tree/master/activemodel/lib/active_model)). The biggest benefit here is the availability of `.validates` on your class to validate attributes and capture invalid attributes with `#valid?`.

Be aware that validations needing context from the database (i.e. `validates_uniqueness_of`) are not yet supported and will need to be implemented manually.

Database callbacks (i.e. `before_save :create_token!`) are not fully functional yet, so stay tuned.

## Available Methods
  * **.create(attributes_hash)**
    - Intantiate *and* save a new object with the given attributes
    - Assigns the returned object it's new ID
    ```
      Post.create(content: 'Amazing post content', author: 'Cullen Jett')
    ```

  * **.find(id)**
    - Query for a specific QuickBase record by it's ID
    - Returns a single object
    ```
      Post.find(params[:id])
    ```

  * **.where(attributes_hash)**
    - Query QuickBase by any field name defined in your class' field mapping
    - Returns an array of objects
    ```
      Post.where(author: 'Cullen Jett').first
    ```

    - Multiple field_name/value pairs are joined with 'AND'
    ```
      Post.where(id: 1, author: 'Cullen Jett')
      # {'3'.EX.'1'}AND{'8'.EX.'Cullen Jett'}
    ```

    - Values in an array are joined with 'OR'
    ```
      Post.where(author: ['Cullen Jett', 'Socrates')
      # {'8'.EX.'Socrates'}OR{'8'.EX.'Socrates'}
    ```

    - To use a comparitor other than 'EX' pass the value as another hash with the key as the comparitor
    ```
      Post.where(author: {XEX: 'Cullen Jett'})
      # {'8'.XEX.'Cullen Jett'}
    ```

  * **.qid(id)**
    - Accepts a QID (QuickBase report ID)
    - Returns an array of objects
    ```
      Post.qid(1)
    ```

  * **#save**
    - Creates a new record in QuickBase for objects that don't have an ID *or* edits the corresponding QuickBase record if the object already has an ID
    - Returns the object (if #save created a record in QuickBase the the returned object will now have an ID)
    - Uses API_ImportFromCSV under the hood.
    ```
      @post = Post.new(content: 'Amazing post content', author: 'Cullen Jett')
      @post.save # => <Post: @id: 1, @content: 'Amazing post content', @author: 'Cullen Jett'

      @post.author = 'Socrates'
      @post.save # => <Post: @id: 1, @content: 'Amazing post content', @author: 'Socrates'
    ```

  * **#delete**
    - Delete the corresponding record in QuickBase
    - It returns the object's ID if successful or `false` if unsuccessful
    ```
      @post = Post.find(1)
      @post.delete
    ```

  * **#update_attributes(attributes_hash)**
    - ** IMPORTANT: Updates *and* saves the object with the new attributes**
    - Returns the object
    ```
      @post = Post.where(author: 'Cullen Jett').first
      @post.update_attributes(author: 'Socrates', content: 'Something enlightening...') # => <Post: @id: 1, @author: 'Socrates', @content: 'Something enlightening...'
    ```

  * **#assign_attributes(attributes_hash)**
    - Only changes the objects attributes in memory (i.e. does not save to QuickBase)
    - Useful for assigning multiple attributes at once, otherwise you could use the field name's attr_accessor to change a single attribute.
    - Returns the object
    ```
      @post = Post.where(author: 'Cullen Jett').first
      @post.assign_attributes(author: 'Socrates', content: 'Something enlightening...')
      @post.save
    ```

## Testing
Unfortunately you will not be able to run the test suite unless you have access to the QuickBase application used as the test database *or* you create your own QuickBase app to test against that mimics the test fakes. Eventually the test calls will be stubbed out so anyone can test it, but I've got stuff to do -- pull requests are welcome :)

As of now the tests serve more as documentation for those who don't have access to the testing QuickBase app.

If you're lucky enough to work with me then I can grant you access to the app and you can run the suite until your fingers bleed. You'll just need to modify `spec/quickbase_record_config.rb` to use your own credentials.

## Contributing

1. Fork it ( https://github.com/[my-github-username]/quickbase_record/fork )
2. Create your feature branch (`git checkout -b my-new-feature`)
3. Commit your changes (`git commit -am 'Add some feature'`)
4. Push to the branch (`git push origin my-new-feature`)
5. Create a new Pull Request
