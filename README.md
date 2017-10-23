# RailsCasts [Episode #396: Importing CSV and Excel](http://railscasts.com/episodes/396-importing-csv-and-excel)

Requires Ruby 1.9.2 or higher.

Re-created on ruby 2.2.6p396 (2016-11-15 revision 56800) [x64-mingw32]

In episode 362 we showed how to export database records to a CSV or Excel file. Since then there have been a number of requests for an episode showing how to import records from these types of files so that’s what we’ll cover in this episode.

We’ll do this by adding a form to the bottom of this page that will allow the user to upload a file containing records. When the form is submitted the file will be parsed and the records added to the database. We’ll add the form at the bottom of the view template


_/app/views/products/index.html.erb_

```ruby
<h2>Import Products</h2>
<%= form_tag import_products_path, multipart: true do %>
  <%= file_field_tag :file %>
  <%= submit_tag "Import" %>
<% end %>
```
We don’t have an object to handle the importing right now so we’ve used form_tag instead of form_for. The form will be submitted to a new import action on the ProductsController and note that we’ve set the multipart option so that the form can handle file uploads. We’ll need to set up the new path in the routes file so we’ll do that now.
_/config/routes.rb_

```ruby
Store::Application.routes.draw do
  resources :products do
    collection { post :import }
  end
  root to: 'products#index'
end
```

Now in the ProductsController we’ll add the import action. This should take the uploaded file then import its data into our database. The file will be uploaded into a file parameter and Rails will store the uploaded file temporarily in the file system while it’s processed. This means that we don’t have to use CarrierWave or Paperclip to work with uploaded files. In this action we’ll pass the uploaded file to a new import method on the Product model then redirect back to the home page.

_/app/controllers/products_controller.rb_

```ruby
def import
  Product.import(params[:file])
  redirect_to root_url, notice: "Products imported."
end
```
#### Importing CSV Data

Now we can focus on the model and the import behaviour. We already have some code in this class from the for exporting CSV data so we’ll focus on importing CSV data before we try handling Excel files. Our app’s config file already has the line require 'csv' so that we can use Ruby’s built-in CSV library.

_/app/models.product.rb_

```ruby
def self.import(file)
  CSV.foreach(file.path, headers: true) do |row|
    Product.create! row.to_hash
  end
end
```

The import method is shown above. In it we call CSV.foreach and pass it the path to the file. This will yield to the block for each line of data that’s found. We’ve used the headers option so the first line of data will be expected to hold each column’s name which will be used to name the data. We then create a product by passing row.to_hash. As long as the column names map to attributes in Product a new record will be created for each row. We’ll try this with a simple CSV file.
products.csv

```csv
name,price,released_on
Christmas Music Album,12.99,2012-12-06
Unicorn Action Figure,5.85,2012-12-06
```
When we upload this file through our new form and submit it the new products appear in the list.

#### Modifying Existing Records

It would be useful if we had an id column in our data that could be used to update an existing record instead of adding a new one. This way we could download an CSV file, modify the products in it then upload it to change multiple products at once. If we download our existing products we’ll end up with this CSV data.
products.csv

```csv
id,name,released_on,price,created_at,updated_at
4,Acoustic Guitar,2012-12-26,1025.0,2012-12-29 18:23:40 UTC,2012-12-29 18:23:40 UTC
5,Agricola,2012-10-31,45.99,2012-12-29 18:23:40 UTC,2012-12-29 18:23:40 UTC
6,Christmas Music Album,2012-12-06,12.99,2012-12-29 20:55:29 UTC,2012-12-29 20:55:29 UTC
2,Red Shirt,2012-10-04,12.49,2012-12-29 18:23:40 UTC,2012-12-29 18:23:40 UTC
1,Settlers of Catan,2012-10-01,34.95,2012-12-29 18:23:40 UTC,2012-12-29 18:23:40 UTC
3,Technodrome,2012-12-22,27.99,2012-12-29 18:23:40 UTC,2012-12-29 18:23:40 UTC
7,Unicorn Action Figure,2012-12-06,5.85,2012-12-29 20:55:29 UTC,2012-12-29 20:55:29 UTC
```

To get this to work we’ll need to change the way that the products are imported. Instead of creating a product for each row of data we’ll try to find one based on the value in the id column. We’ll use find_by_id so that nil is returned if a matching record isn’t found and in this case we’ll create a new record. Next we’ll set the product’s attributes based on the data in the row and as this might include attributes we don’t want to set attributes for such as the id we’ll update off the only the attributes that are listed in the model’s attr_accessible list.

_/app/models/product.rb_

```ruby
def self.import(file)
  CSV.foreach(file.path, headers: true) do |row|
    product = find_by_id(row["id"]) || new
    product.attributes = row.to_hash.slice(*accessible_attributes)
    product.save!
  end
end
```

We’ve edited our CSV file now and altered a couple of the products’ names. If we upload this file we should see these changes but no new products added.

This has worked. The two products we renamed have their new names showing and no new records have been added.

#### Importing Excel Spreadsheets

Now that we have CSV files working how can we import an Excel file? There are several gems available that handle importing from Excel; in this episode we’ll use the Roo gem as it provides a standardized interface for accessing a variety of spreadsheet formats, including Excel and CSV. The gem is installed in the usual way, by adding it to our application’s gemfile and running bundle.

_/Gemfile_

```rudy
gem 'roo'
```

It’s also necessary to modify our application’s config file to require the iconv library. Unfortunately doing this adds some warnings every time we start up our Rails application so hopefully the gem will move away from using this soon.

_/config/application.rb_

```ruby
require 'iconv'
```
Now that we have Roo installed we can use it to import product records from a spreadsheet. The first thing we need to do is get a spreadsheet from Roo. Doing this can be a little complicated so we’ll do it in a separate method that we’ll call open_spreadsheet and write shortly. A Roo spreadsheet has a row method which returns an array of values from that row. The first row will contain the header details so we’ll fetch that first. We’ll then loop through the other rows and fetch each one’s data, calling last_row on our spreadsheet object to get the total number of rows.

Next comes the tricky part. Fetching each row returns an array of values but we need to convert that to a hash, with the header columns as the keys. To do this we create an array of the header and the current row and call transpose on that to create an array of arrays, each one of which contains the header name and the appropriate value for the current row. Finally we convert this to a hash which gives us an object similar to the one we got from the CSV library.

_/app/models/product.rb_

```ruby
def self.import(file)
  spreadsheet = open_spreadsheet(file)
  header = spreadsheet.row(1)
  (2..spreadsheet.last_row).each do |i|
    row = Hash[[header, spreadsheet.row(i)].transpose]
    product = find_by_id(row["id"]) || new
    product.attributes = row.to_hash.slice(*accessible_attributes)
    product.save!
  end
end
```
Next we need to define open_spreadsheet method. This will build up a different Roo spreadsheet depending on the file extension. We use original_filename on the uploaded file because it’s stored in a temporary file which doesn’t have an extension. Note that the current master branch of Roo has the class names under a Roo namespace so that when a new version is released we’ll need to use, for example, Roo::Excel instead of just Excel. The third option, :ignore, tells Roo not to raise an exception if the file extension doesn’t match the type.

_/app/models/product.rb_

```ruby
def self.open_spreadsheet(file)
  case File.extname(file.original_filename)
  when '.csv' then Csv.new(file.path, nil, :ignore)
  when '.xls' then Excel.new(file.path, nil, :ignore)
  when '.xlsx' then Excelx.new(file.path, nil, :ignore)
  else raise "Unknown file type: #{file.original_filename}"
  end
end
```

We have an Excel file with the correct columns and a couple of product records in xlsx format so we’ll upload it through our form and see if that works.
