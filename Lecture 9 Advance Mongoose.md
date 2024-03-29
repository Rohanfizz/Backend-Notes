### Querying for data in the DB
- query through url `/posts?likes=5&active=true` and access using `req.query`
- Querying in find function
```js
const post = await BlogPostModel.find({
	likes : 5,
	active: true
})
```
- Chaining method - in this we use methods like equals(), lt(), lte, gt, gte, limit 
```js
const posts = await BlogPostModel.find().where('likes').equals(5).where('active').equals(true)
```

### Advance features like sort, pagination, limit and fields
- For advance features, we will need to work with `Query()` object which is returned if we dont `await` the find method. 
	- For example, `const blogPosts = await BlogPostModel.find()` if we `await` the query will be executed and all the results will be returned of the query. But if we don't await, a `Query()` object will be returned.
	- [Query object docs](https://mongoosejs.com/docs/api/query.html#Query())
- Therefore to apply advance features like pagination, sorting etc. we can't await the find() method from the get go as we need to play with the `Query()` object.
```js
const queryObj = {...req.query} // Cloning an object in es6
const excludedFields = ['page','sort','limit','fields']
excludedFields.forEach( field => delete queryObj[field])

const query = BlogpostModel.find(queryObj)// Notice no await here
// We will play with query obj here
//...
//...
const blogPosts = await query;
```

- In req.query, we are only having key value pairs, i.e. we can match one field with a value if that is equal or not. But in our query we want advanced features like gte, lte etc.
- Actual filters in mongoose look like this = 
```js
{active: true, likes: { $gte : 12 }}
```
if you use query like this = `/blogPost/likes[gte]=5&active=true`
output of req.query will look like this-
```js
{active: true, likes: { gte : 12 }}
```
Notice that its only missing "$" sign in front of `gte`
We will use Regex expression to replace gte,gt,lte,lt with `$gte, $gt,$lte,$lt`
```js
let queryStr = JSON.stringify(queryObj)
quertStr = queryStr.replace(/\b(gte|gt|lte|lt)\b/g,match => `$${match}`)
query = Blogpost.find(JSON.parse(queryStr))
```

### Sorting
for sorting, our query url will look like this `/blogPost/sort=likes` and for descending, `/blogpost/sort=-likes`
- On the query object, we will simply do this
```js
let query = Blogpost.find(JSON.parse(queryStr)) // from above code
if(req.query.sort){
	query = query.sort(req.query.sort)
}
```
- If you want sorting on multiple fields just add space like, `query.sort("likes comments")`
- Also if no sorting is applied, sorting should happen based on createdAt field but in descending order
```js
let query = Blogpost.find(JSON.parse(queryStr)) // from above code
if(req.query.sort){
	const sortBy = req.query.sort.split(',').join(' ')
	query = query.sort(req.query.sort)
}else{
	query = query.sort('-createdAt')
}
```

### Field limiting (Projecting)
- Projecting is the selection of some fields to be sent and not sending all fields
- Its a good idea to just request what is required. in postman we will create request like this
- `/blogpost/fields=description,likes`
- For mongoose, query should look like this - `query = query.select("likes description")`
```js
if (req.query.fields) {
  const fields = req.query.fields.split(',').join(' ');
  query = query.select(fields);
} else {
  query = query.select('-__v');//not including but excluding
}
```

### Pagination
- We will have 2 fields in our req.query, `blogpost/page=2&limit=10`
- According to the pageNumber and limit, we will have to skip (page-1) * limit number of results
```js
// 4) Pagination
const page = req.query.page * 1 || 1;
const limit = req.query.limit * 1 || 100;
const skip = (page - 1) * limit;

query = query.skip(skip).limit(limit);

// error handling in pagination
if (req.query.page) {
  const numPosts = await BlogPosts.countDocuments();
  if (skip >= numPosts) throw new Error('This page does not exist!');
}
```

### Refactoring
Making class for api features and passing query object and req.query

1. create a class file
```js
class APIFeatures {
  constructor(query, queryString) {
    this.query = query;
    this.queryString = queryString;
  }

  filter() {
    const queryObj = { ...this.queryString };
    const excludedFields = ['page', 'sort', 'limit', 'fields'];
    excludedFields.forEach((el) => delete queryObj[el]);

    // 1B) ADVANCED FILTERING WITH SUPPORT OF lt,gt,gte etc in query url
    let queryStr = JSON.stringify(queryObj);
    queryStr = queryStr.replace(/\b(gte|gt|le|lte)\b/g, (match) => `$${match}`);

    this.query.find(JSON.parse(queryStr));
    return this;
  }
// similar for sort(),limitFields(),paginate()
}
module.exports = APIFeatures;
```
2. import it
```js
const APIFeatures = require('../utils/apiFeatures');
```
3. Use it
```js
 const features = new APIFeatures(Model.findById(req.params.id),req.query)
            .filter()
            .sort()
            .limitFields()
            .paginate();
const document = await features.query;
```
