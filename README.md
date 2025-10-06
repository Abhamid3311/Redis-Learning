# Redis

Install Redis: npm install redis

## Redis Setup File :

src/config/redis.ts

```
import { createClient } from 'redis';

export const redisClient = createClient({
  url: process.env.REDIS_URL || 'redis://127.0.0.1:6379',
});

redisClient.on('error', (err) => console.error('Redis Client Error:', err));

export const connectRedis = async () => {
  if (!redisClient.isOpen) await redisClient.connect();
  console.log('✅ Redis connected');
};
```

- Start Redis Server: redis-server
- Redis only Need to setup in Service Part cause it work with DB.
- Redis Main Concepts:
  - Get Data from DB ( Also Stored in Redis)
  - Again get--> It will not go to DB, It take data from Chech of Redis (Super fast)
  - If Something Update it will Invalidate Old data

## generate Cache key :

```
const generateCacheKey=(req)=>{
    const baseUrl= req.path.replace(/^\/+|\/+$/g, '').replace(/\//g,":");
    const params= req.query;
    const sortedParams= Object.keys(params).sort()
                        .map((key)=> `${key}=${params[key]}).join('&);

    return sortedParams ? `${baseUrl}: ${sortedParams}`:baseUrl;
}
```

## Redis Main Functionality:

```
import { redisClient } from '../../config/redis';

// Key of Get All Post: Will set All data on this key:value pair
const allPostsKey = 'posts:all';
const postKey = (id: string) => `post:${id}`;

// Alternate Approch for generate dynamic key
const key = generateCacheKey(req)


// Get all posts (cache-aside)
export const getAllPosts = async () => {
  const cached = await redisClient.get(allPostsKey);
  if (cached) return JSON.parse(cached);

  const posts = await Post.find().sort({ createdAt: -1 }).lean();
  await redisClient.setEx(allPostsKey, 60, JSON.stringify(posts));
  return posts;
};

// Get single post (cache-aside) by pass a ID
export const getPostById = async (id: string) => {
  const cached = await redisClient.get(postKey(id));
  if (cached) return JSON.parse(cached);

  const post = await Post.findById(id).lean();
  if (post) await redisClient.setEx(postKey(id), 300, JSON.stringify(post));
  return post;
};

//  For Post, Update, delete We just need to Invalidate old Cache by Sending this After Operations

 await redisClient.del(allPostsKey);
  await redisClient.del(postKey(id));

```

## Full Redis Setup in Service Page with MongoDB:

```
import { Post } from './post.model';
import { IPost } from './post.interface';
import { redisClient } from '../../config/redis';

const allPostsKey = 'posts:all';
const postKey = (id: string) => `post:${id}`;

// Get all posts (cache-aside)
export const getAllPosts = async () => {
  const cached = await redisClient.get(allPostsKey);
  if (cached) return JSON.parse(cached);

  const posts = await Post.find().sort({ createdAt: -1 }).lean();
  await redisClient.setEx(allPostsKey, 60, JSON.stringify(posts));
  return posts;
};

// Get single post (cache-aside)
export const getPostById = async (id: string) => {
  const cached = await redisClient.get(postKey(id));
  if (cached) return JSON.parse(cached);

  const post = await Post.findById(id).lean();
  if (post) await redisClient.setEx(postKey(id), 300, JSON.stringify(post));
  return post;
};

// Create post + invalidate cache
export const createPost = async (data: IPost) => {
  const post = await Post.create(data);
  await redisClient.del(allPostsKey);
  return post;
};

// Update post + invalidate cache
export const updatePost = async (id: string, data: Partial<IPost>) => {
  const updated = await Post.findByIdAndUpdate(id, data, { new: true });
  await redisClient.del(allPostsKey);
  await redisClient.del(postKey(id));
  return updated;
};

// Delete post + invalidate cache
export const deletePost = async (id: string) => {
  await Post.findByIdAndDelete(id);
  await redisClient.del(allPostsKey);
  await redisClient.del(postKey(id));
};
```

## Redis Policy Set:

Which data We Should Saved? Limit Data: We have some famous step . any of we can use:

```
CONFIG SET maxmemory 100mb // Saved Max 100 MB, After that will give error

CONFIG SET maxmemory-policy allkeys-lru // List Recently used ( Store Recent data in chech, Removed old data)

CONFIG SET maxmemory-policy allkeys-lfu // List Frequently used ( Store Frequently data in chech, Removed other data)


```
