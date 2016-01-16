[![Coverage](http://gocover.io/_badge/github.com/NickSardo/jsonpath)](http://gocover.io/github.com/NickSardo/jsonpath)
# jsonpath  
  
jsonpath is used to pull values out of a JSON document without unmarshalling the string into an object.  At the loss of post-parse random access and conversion to primitive types, you gain faster return speeds and lower memory utilization.  If the value you want is located near the start of the json, the evaluator will terminate after reaching and recording its destination.  
  
The evaluator can be initialized with several paths, so you can retrieve multiple sections of the document with just one scan.  Naturally, when all paths have been reached, the evaluator will early terminate.  
  
For each value returned by a path, you'll also get the keys & indexes needed to reach that value.  Use the `keys` flag to view this in the CLI.  The Go package will return an `[]interface{}` of length `n` with indexes `0 - (n-2)` being the keys and the value at index `n-1`.  
  
### CLI   
```shell
go get github.com/NickSardo/jsonpath/cli/jsonpath
cat yourData.json | jsonpath -k -p '$.Items[*].title+'
```

##### Usage  
```shell
-f, --file="": Path to json file  
-j, --json="": JSON text  
-k, --keys=false: Print keys & indexes that lead to value  
-p, --path=[]: One or more paths to target in JSON
```

  
### Go Package  
go get github.com/NickSardo/jsonpath  
 
```go
paths, err := jsonpath.ParsePaths(pathStrings ...string) {
```  

```go
eval, err := jsonpath.EvalPathsInBytes(json []byte, paths) 
// OR
eval, err := jsonpath.EvalPathsInReader(r io.Reader, paths)
```

then  
```go  
for {
	if result, ok := eval.Next(); ok {
		fmt.Println(result.Pretty(true)) // true -> show keys in pretty string
	} else {
		break
	}
}
if eval.Error != nil {
	return eval.Error
}
```  

`eval.Next()` will traverse JSON until another value is found.  This has the potential of traversing the entire JSON document in an attempt to find one.  If you prefer to have more control over traversing, use the `eval.Iterate()` method.  It will return after every scanned JSON token and return `([]*Result, bool)`.  This array will usually be empty, but occasionally contain results.  
     
### Path Syntax  
All paths start from the root node `$`.  Similar to getting properties in a JavaScript object, a period `.title` or brackets `["title"]` are used.  
  
Syntax|Meaning|Examples
------|-------|-------
`$`|root of doc|  
`.`|property selector |`$.Items`
`["abc"]`|quoted property selector|`$["Items"]`
`*`|wildcard property name|`$.*` 
`[n]`|Nth index of array|`[0]` `[1]`
`[n:m]`|Nth index to m-1 index (same as Go slicing)|`[0:1]` `[2:5]`
`[n:]`|Nth index to end of array|`[1:]` `[2:]`
`[*]`|wildcard index of array|`[*]`
`+`|get value at end of path|`$.title+`
`?(expression)`|where clause (expression can reference current json node with @)|`?(@.title == "ABC")`
  
  
Expressions  
- paths (that start from current node `@`)
- numbers (integers, floats, scientific notation)
- mathematical operators (+ - / * ^)
- numerical comparisos (< <= > >=)
- logic operators (&& || == !=)
- parentheses `(2 < (3 * 5))`
- static values like (`true`, `false`)
- `@.value > 0.5`

Example: this will only return tags of all items that match this expression.
`$.Items[*]?(@.title == "A Tale of Two Cities").tags`  

   
Example: 
```javascript
{  
	"Items":   
		[  
			{  
				"title": "A Midsummer Night's Dream",  
				"tags":[  
					"comedy",  
					"shakespeare",  
					"play"  
				]  
			},{  
				"title": "A Tale of Two Cities",  
				"tags":[  
					"french",  
					"revolution",  
					"london"  
				]  
			}  
		]  
} 
```
	
Example Paths:   
*CLI*  
```shell
jsonpath --file=example.json --path='$.Items[*].tags[*]+' --keys
```   
"Items"	0	"tags"	0	"comedy"  
"Items"	0	"tags"	1	"shakespeare"  
"Items"	0	"tags"	2	"play"  
"Items"	1	"tags"	0	"french"  
"Items"	1	"tags"	1	"revolution"  
"Items"	1	"tags"	2	"london"  
  
*Paths*  
`$.Items[*].title+`   
... "A Midsummer Night's Dream"   
... "A Tale of Two Cities"   
  
`$.Items[*].tags+`    
... ["comedy","shakespeare","play"]  
... ["french","revolution","london"]  
  
`$.Items[*].tags[*]+`  
... "comedy"  
... "shakespeare"  
... "play"  
... "french"  
... "revolution"  
...  "london"  
  
... = keys/indexes of path  

### Performance

	BenchmarkUnmarshalMix-8     	     500	   3358953 ns/op	 1262149 B/op	   22784 allocs/op
	BenchmarkDecodeMix-8        	    1000	   1149598 ns/op	     644 B/op	      24 allocs/op
	BenchmarkSliceMix-8         	    2000	    729137 ns/op	     896 B/op	       1 allocs/op
	BenchmarkReaderMix-8        	     500	   2513132 ns/op	     896 B/op	       1 allocs/op
	BenchmarkUnmarshalDigits-8  	   10000	    115365 ns/op	   43448 B/op	    1315 allocs/op
	BenchmarkDecodeDigits-8     	   50000	     24255 ns/op	      48 B/op	       1 allocs/op
	BenchmarkSliceDigits-8      	   50000	     43576 ns/op	     896 B/op	       1 allocs/op
	BenchmarkReaderDigits-8     	   20000	     67710 ns/op	     896 B/op	       1 allocs/op
	BenchmarkUnmarshalStrings-8 	   30000	     57586 ns/op	   22584 B/op	     630 allocs/op
	BenchmarkDecodeStrings-8    	  100000	     16210 ns/op	      48 B/op	       1 allocs/op
	BenchmarkSliceStrings-8     	  100000	     13828 ns/op	     896 B/op	       1 allocs/op
	BenchmarkReaderStrings-8    	   30000	     40430 ns/op	     896 B/op	       1 allocs/op
	BenchmarkUnmarshalLiterals-8	   30000	     41388 ns/op	   16912 B/op	     262 allocs/op
	BenchmarkDecodeLiterals-8   	  100000	     17727 ns/op	      48 B/op	       1 allocs/op
	BenchmarkSliceLiterals-8    	  100000	     22474 ns/op	     896 B/op	       1 allocs/op
	BenchmarkReaderLiterals-8   	   30000	     55023 ns/op	     896 B/op	       1 allocs/op
	BenchmarkUnmarshalArrays-8  	   10000	    116467 ns/op	   32456 B/op	    1014 allocs/op
	BenchmarkDecodeArrays-8     	  100000	     20497 ns/op	      48 B/op	       1 allocs/op
	BenchmarkSliceArrays-8      	   50000	     26537 ns/op	   12544 B/op	       4 allocs/op
	BenchmarkReaderArrays-8     	   30000	     46001 ns/op	   12544 B/op	       4 allocs/op
	BenchmarkUnmarshalObjects-8 	   10000	    102506 ns/op	   76048 B/op	     643 allocs/op
	BenchmarkDecodeObjects-8    	   50000	     25513 ns/op	       0 B/op	       0 allocs/op
	BenchmarkSliceObjects-8     	  100000	     19682 ns/op	    5888 B/op	       3 allocs/op
	BenchmarkReaderObjects-8    	   30000	     52160 ns/op	    5888 B/op	       3 allocs/op
	BenchmarkStdUnmarshalLarge-8	       1	6120026114 ns/op	1185057216 B/op	36537963 allocs/op
	BenchmarkStdLibDecodeLarge-8	       1	1919992027 ns/op	536869232 B/op	      34 allocs/op
	BenchmarkSliceLexerLarge-8  	       1	1655279052 ns/op	     896 B/op	       1 allocs/op
	BenchmarkReaderLexerLarge-8 	       1	3898017338 ns/op	     896 B/op	       1 allocs/op
