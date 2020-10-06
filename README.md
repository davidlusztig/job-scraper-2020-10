Job Board Scraping with Rails
Build a scheduled job scraper with Ruby on Rails and Heroku
Chris I.
Chris I.
Mar 29 · 5 min read
Image for post
Image for post
Photo by Donald Giannatti on Unsplash

Job-related data is one of my favorite to play with. It’s fun to parse requirements and analyze changes across time, companies and geography.

While there are some great public databases with job-related information, it’s also pretty available to scrape.

We’re going to build a job board scraper in Ruby on Rails that will automatically run once per day, hosted on the free tier of Heroku.
Setup

Assuming you have Rails installed, create a new rails app and cd into its root directory.

$ rails new scraper-2020-03
$ cd scraper-2020-03/

Then modify Gemfile, where ruby dependencies are set. Comment out this line.

# gem 'sqlite3'

And add this line.

gem 'pg'

Then run this to update installed packages.

$ bundle install

We’ve just replaced Sqlite with Postgres.
Configure the Database

Open /config/database.yml and update the file so it looks like this.

default: &default
  adapter: postgresql
  pool: 5
  timeout: 5000

development:
  <<: *default
  database: scraper_development

test:
  <<: *default
  database: scraper_test

production:
  <<: *default
  url: <%= ENV['DATABASE_URL'] %>

This important parts here are setting development’s db name, database: scraper_development, and production’s url, url: <%= ENV[‘DATABASE_URL’] %>.

The latter is important for when we deploy to Heroku. Heroku sets the URL for it’s Postgres addon to the environment variable, DATABASE_URL by default.

Now create a database locally. It will be named what we set above.

$ rake db:create

Create the ORM Model

Generate a migration file. This is where we specify what changes we want to make to the database.

$ rails g migration CreateJobs

Navigate to the migration file just generated. Mine is db/migrate/20200328142555_create_jobs.rb, timestamps will vary.

Edit it to look like below.

class CreateJobs < ActiveRecord::Migration[5.0]
  def change
    create_table :jobs do |t|
      t.string :location
      t.string :team
      t.string :job_title
      t.string :url

      t.timestamps
    end
  end
end

We’re creating a table called jobs with several string columns to store the scraped information. A primary id will be added by default. t.timestamps will add created_at and updated_at columns which are populated automatically when you insert records into the database.

Create a file called job.rb in /app/models/. This is our model. It should look like this.

class Job < ApplicationRecord
end

Create a Rake Task

In /lib/tasks create a new file called scrape.rake. Rake tasks in Rails are commands used to automate administrative tasks which can be run from the command line. If you want to write code and schedule when it runs, this is where to put it.

Update scrape.rake.

task scrape: :environment do
  puts 'HERE'

  require 'open-uri'

  URL = 'https://jobs.lever.co/stackadapt'

  doc = Nokogiri::HTML(open(URL))

  postings = doc.search('div.posting')

  postings.each do |p|
    job_title = p.search('a.posting-title > h5').text
    location = p.search('a.posting-title > div > span')[0].text
    team = p.search('a.posting-title > div > span')[1].text
    url = p.search('a.posting-title')[0]['href']

    # skip persisting job if it already exists in db
    if Job.where(job_title:job_title, location:location, team:team, url:url).count <= 0
      Job.create(
        job_title:job_title,
        location:location,
        team:team,
        url:url)

      puts 'Added: ' + (job_title ? job_title : '')
    else
      puts 'Skipped: ' + (job_title ? job_title : '')
    end

  end
end

Navigating html with Nokogiri (Ruby’s html parsing library) is beyond our scope but if you open the webpage and inspect, you can follow along how I parsed the DOM tree.
Run Locally

The app is all set up. Running locally is as easy as this.

$ rake scrape

In my local database, I can now see the data. Consider dbeaver if you don’t have a good tool to peer into your local database.
Image for post
Image for post

Our only problem is we don’t want to manually run it every day. Fix that by deploying to Heroku and scheduling the task.
Deploy to Production

Create or login to your Heroku account in your browser.

Then type below into your command line. It will use your browser to authenticate.

$ heroku login

Run the following command to create your app on Heroku.

$ heroku create

Heroku will output the name it gives your app in response. Mine is called fast-garden-42976.

Now set up git locally.

$ git init
$ git add . -A
$ git commit -m 'The first and last commit'

Add your app with the name Heroku gave you (not mine) so we can deploy.

$ heroku git:remote -a fast-garden-42976

And deploy.

$ git push heroku master

When that finishes, migrate the database on Heroku.

$ heroku run rake db:migrate

Configure Scheduler in Production

Test it on production by manually running the command.

$ heroku run rake scrape

Now we just need to setup the rake task as a scheduled job.

In the Heroku web console, click on “resources”.
Image for post
Image for post

Start typing “schedule” and provision Heroku Scheduler.
Image for post
Image for post

Click on the scheduler you just added and click “Create Job”.
Image for post
Image for post

Configure the job, rake scrape to run once a day at midnight.

Be kind, don’t scrape sites more than necessary.

And voila. Your app will now add any new jobs once every 24 hours.

When you want to access this data, consider an app like Postico which allows directly connecting to the remote database in Heroku. Alternatively, you can download the database from Heroku and import it locally.
Conclusion

This was a quick and easy scraping app. We grabbed some preliminary information about each job and dumped it into a database.

If you wanted to take this to the next level, there are a few more things you may want to add.

    Open the stored URL in order to scrape the job’s description.
    Add another column to the database to store company name if scraping multiple job boards.

I hope that was informative and I’m happy to answer questions in the comments.
