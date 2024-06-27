---
layout: default
---

Text can be **bold**, _italic_, ~~strikethrough~~ or `keyword`.

[Link to another page](./job-2024-06-26T10.34-867f350/results.html).

There should be whitespace between paragraphs.

There should be whitespace between paragraphs. We recommend including a README, or a file with information about your project.

# CBD （CXL Block Device）

As shared memory is supported in CXL3.0 spec, we can transfer data via CXL shared memory. CBD means CXL block device, it use CXL shared memory 
to transfer command and data to access block device in different host, as shown below:

```js
     +-------------------------------+                               +------------------------------------+
     |          node-1               |                               |              node-2                |
     +-------------------------------+                               +------------------------------------+
     |                               |                               |                                    |
     |                       +-------+                               +---------+                          |
     |                       | cbd0  |                               | backend0+------------------+       |
     |                       +-------+                               +---------+                  |       |
     |                       | pmem0 |                               | pmem0   |                  v       |
     |               +-------+-------+                               +---------+----+     +---------------+
     |               |    cxl driver |                               | cxl driver   |     |   /dev/sda    |
     +---------------+--------+------+                               +-----+--------+-----+---------------+
                              |                                            |
                              |                                            |
                              |        CXL                         CXL     |
                              +----------------+               +-----------+
                                               |               |
                                               |               |
                                               |               |
                                           +---+---------------+-----+
                                           |   shared memory device  |
                                           +-------------------------+
```

any read/write to cbd0 on node-1 will be transferred to node-2 /dev/sda. It works similar with
nbd (network block device), but it transfer data via CXL shared memory rather than network.

## Linux kernel module

The kernel module for cbd is in going to linux kernel tree.

## cbd userspace tool

## cbd test suites

### test results

```js
// Javascript code with syntax highlighting.
var fun = function lang(l) {
  dateformat.i18n = require('./lang/' + l)
  return true;
}
```

```ruby
# Ruby code with syntax highlighting
GitHubPages::Dependencies.gems.each do |gem, version|
  s.add_dependency(gem, "= #{version}")
end
```

#### Header 4

*   This is an unordered list following a header.
*   This is an unordered list following a header.
*   This is an unordered list following a header.

##### Header 5

1.  This is an ordered list following a header.
2.  This is an ordered list following a header.
3.  This is an ordered list following a header.

###### Header 6

| head1        | head two          | three |
|:-------------|:------------------|:------|
| ok           | good swedish fish | nice  |
| out of stock | good and plenty   | nice  |
| ok           | good `oreos`      | hmm   |
| ok           | good `zoute` drop | yumm  |

### There's a horizontal rule below this.

* * *

### Here is an unordered list:

*   Item foo
*   Item bar
*   Item baz
*   Item zip

### And an ordered list:

1.  Item one
1.  Item two
1.  Item three
1.  Item four

### And a nested list:

- level 1 item
  - level 2 item
  - level 2 item
    - level 3 item
    - level 3 item
- level 1 item
  - level 2 item
  - level 2 item
  - level 2 item
- level 1 item
  - level 2 item
  - level 2 item
- level 1 item

### Small image

![Octocat](https://github.githubassets.com/images/icons/emoji/octocat.png)

### Large image

![Branching](https://guides.github.com/activities/hello-world/branching.png)


### Definition lists can be used with HTML syntax.

<dl>
<dt>Name</dt>
<dd>Godzilla</dd>
<dt>Born</dt>
<dd>1952</dd>
<dt>Birthplace</dt>
<dd>Japan</dd>
<dt>Color</dt>
<dd>Green</dd>
</dl>

```
Long, single-line code blocks should not wrap. They should horizontally scroll if they are too long. This line should be long enough to demonstrate this.
```

```
The final element.
```
