You need to have CMake and `pkg-config` installed on your system to be able to build the included version of `libgit2`. On OS X, after installing [Homebrew](http://brew.sh/), you can get CMake with:
```bash
$ brew install cmake
```

If you want to build Rugged with HTTPS and SSH support, check out the list of optional [libgit2 dependencies](https://github.com/libgit2/libgit2#optional-dependencies).
### Use the system provided libgit2

By default, Rugged builds and uses a bundled version of libgit2. If you
want to use the system library instead, you can install rugged as follows:

```
gem install rugged -- --use-system-libraries
```

Or if you are using bundler:

```
bundle config build.rugged --use-system-libraries
bundle install
```

However, note that Rugged does only support specific versions of libgit2.

repo.head_unborn?
# => #<Rugged::Reference:2228467240 {name: "refs/heads/master", target:  #<Rugged::Commit:2228467250 {message: "helpful message", tree: #<Rugged::Tree:2228467260 {oid: 5d6f29220a0783b8085134df14ec4d960b6c3bf2}>}>
# From the returned ref, you can also access the `name`, `target`, and target SHA:
# => #<Rugged::Commit:2228467250 {message: "helpful message", tree: #<Rugged::Tree:2228467260 {oid: 5d6f29220a0783b8085134df14ec4d960b6c3bf2}>}>
ref.target_id
# => "2bc6a70483369f33f641ca44873497f13a15cde5"
index = repo.index
index.read_tree(repo.head.target.tree)
### Blob Objects

Blob objects represent the data in the files of a Tree Object.

```ruby
blob = repo.lookup('e1253910439ea902cf49be8a9f02f3c08d89ac73')
blob.content # => Gives you the content of the blob.
```

#### Streaming Blob Objects

There is currently no way to stream data from a blob, because `libgit2` itself does not (yet) support
streaming blobs out of the git object database. While there are hooks and interfaces for supporting it,
the default file system backend always loads the entire blob contents into memory. 

If you need to access a Blob object through an IO-like API, you can wrap it with the `StringIO` class.
Note that the only advantage here is a stream-compatible interface, the complete blob object will still
be loaded into memory. Below is an example for streaming a Blob using the Sinatra framework:

```ruby
# Sinatra endpoint
get "/blobs/:sha" do
  repo = Rugged::Repository.new(my_repo_path)
  blob = repo.lookup params[:sha]

  headers({
    "Vary" => "Accept",
    "Connection" => "keep-alive",
    "Transfer-Encoding" => "chunked",
    "Content-Type" => "application/octet-stream",
  })

  stream do |out|
    StringIO.new(blob.content).each(8000) do |chunk|
      out << chunk
    end
  end
end
```

### Diffs

There are various ways to get hands on diffs:

```ruby
# Diff between two subsequent commits
diff_commits = commit_object.parents[0].diff(commit_object)

# Diff between two tree objects
diff_trees = tree_object_a.diff(tree_object_b)

# Diff between index/staging and current working directory
diff_index = repository.index.diff

# Diff between index/staging and another diffable (commit/tree/index)
diff_index_diffable = repository.index.diff(some_diffable)
```

When you already have a diff object, you can examine it:

```ruby
# Get patch
diff.patch
=> "diff --git a/foo1 b/foo1\nnew file mode 100644\nindex 0000000..81b68f0\n--- /dev/null\n+++ b/foo1\n@@ -0,0 +1,2 @@\n+abc\n+add line1\ndiff --git a/txt1 b/txt1\ndeleted file mode 100644\nindex 81b68f0..0000000\n--- a/txt1\n+++ /dev/null\n@@ -1,2 +0,0 @@\n-abc\n-add line1\ndiff --git a/txt2 b/txt2\nindex a7bb42f..a357de7 100644\n--- a/txt2\n+++ b/txt2\n@@ -1,2 +1,3 @@\n abc2\n add line2-1\n+add line2-2\n"

# Get delta (faster, if you only need information on what files changed)
diff.each_delta{ |d| puts d.inspect }
#<Rugged::Diff::Delta:70144372137380 {old_file: {:oid=>"0000000000000000000000000000000000000000", :path=>"foo1", :size=>0, :flags=>6, :mode=>0}, new_file: {:oid=>"81b68f040b120c9627518213f7fc317d1ed18e1c", :path=>"foo1", :size=>14, :flags=>6, :mode=>33188}, similarity: 0, status: :added>
#<Rugged::Diff::Delta:70144372136540 {old_file: {:oid=>"81b68f040b120c9627518213f7fc317d1ed18e1c", :path=>"txt1", :size=>14, :flags=>6, :mode=>33188}, new_file: {:oid=>"0000000000000000000000000000000000000000", :path=>"txt1", :size=>0, :flags=>6, :mode=>0}, similarity: 0, status: :deleted>
#<Rugged::Diff::Delta:70144372135780 {old_file: {:oid=>"a7bb42f71183c162efea5e4c80597437d716c62b", :path=>"txt2", :size=>17, :flags=>6, :mode=>33188}, new_file: {:oid=>"a357de7d870823acc3953f1b2471f9c18d0d56ea", :path=>"txt2", :size=>29, :flags=>6, :mode=>33188}, similarity: 0, status: :modified>

# Detect renamed files
# Note that the status field changed from :added/:deleted to :renamed
diff.find_similar!
diff.each_delta{ |d| puts d.inspect }
#<Rugged::Diff::Delta:70144372230920 {old_file: {:oid=>"81b68f040b120c9627518213f7fc317d1ed18e1c", :path=>"txt1", :size=>14, :flags=>6, :mode=>33188}, new_file: {:oid=>"81b68f040b120c9627518213f7fc317d1ed18e1c", :path=>"foo1", :size=>14, :flags=>6, :mode=>33188}, similarity: 100, status: :renamed>
#<Rugged::Diff::Delta:70144372230140 {old_file: {:oid=>"a7bb42f71183c162efea5e4c80597437d716c62b", :path=>"txt2", :size=>17, :flags=>6, :mode=>33188}, new_file: {:oid=>"a357de7d870823acc3953f1b2471f9c18d0d56ea", :path=>"txt2", :size=>29, :flags=>6, :mode=>33188}, similarity: 0, status: :modified>

# Merge one diff into another (mutating the first one)
diff1.merge!(diff2)

# Write a patch into a file (or any other object responding to write)
# Note that the patch as in diff.patch will be written, it won't be applied
file = File.open('/some/file', 'w')
diff.write_patch(file)
file.close
```

---
