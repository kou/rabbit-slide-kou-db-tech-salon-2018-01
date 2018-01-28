require "csv"

require "rabbit/task/slide"

# Edit ./config.yaml to customize meta data

spec = nil
Rabbit::Task::Slide.new do |task|
  spec = task.spec
  spec.files += Dir.glob("images/*.*")
  # spec.files -= Dir.glob("private/**/*.*")
  spec.add_runtime_dependency("rabbit-theme-groonga")
end

desc "Tag #{spec.version}"
task :tag do
  sh("git", "tag", "-a", spec.version.to_s, "-m", "Publish #{spec.version}")
  sh("git", "push", "--tags")
end

crimes_csv = "crimes.csv"
file crimes_csv do |task|
  sh("wget", "-O", task.name,
     "https://data.cityofchicago.org/api/views/ijzp-q8t2/rows.csv?accessType=DOWNLOAD")
end

def sql_escape_string(string)
  escaped_string = string.gsub(/'/, "''")
  "'#{escaped_string}'"
end

crimes_sql = "crimes.sql"
file crimes_sql => crimes_csv do |task|
  File.open(task.name, "w") do |sql|
    sql.puts(<<-SQL)
DROP TABLE IF EXISTS crimes;
CREATE TABLE crimes (
  id int PRIMARY KEY,
  block varchar(512),
  description varchar(512),
  arrest boolean,
  domestic boolean,
  ward int,
  community_area int,
  year int
) ENGINE=Mroonga;
SQL
    CSV.open(task.prerequisites[0], headers: true) do |csv|
      csv.each.each_slice(10000) do |rows|
        sql.puts("INSERT INTO crimes VALUES")
        rows.each_with_index do |row, i|
          sql.puts(",") unless i.zero?
          values = [
            row["ID"],
            sql_escape_string(row["Block"]),
            sql_escape_string(row["Description"]),
            row["Arrest"],
            row["Domestic"],
            row["Ward"] || 0,
            row["Community Area"] || 0,
            row["Year"],
          ].join(", ")
          sql.print("(#{values})")
        end
        sql.puts
        sql.puts(";")
      end
    end
  end
end

task "Generate data"
task :generate => crimes_sql
