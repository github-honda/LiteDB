From: hondachen@hotmail.com
Date: 2020-12-13
Subject: readme-LiteDB.txt

----------
Summary:

https://github.com/github-honda/LiteDB

參考:
  https://github.com/mbdavid/LiteDB
  https://github.com/mbdavid/LiteDB/wiki
  http://www.litedb.org/docs/ v5 文件
  http://www.litedb.org/
  
歡迎來信交流.


----------
20201213

https://dotblogs.com.tw/supershowwei/2018/04/25/230600
https://dotblogs.com.tw/supershowwei/2018/05/07/103920

LiteDB 是一個 Document-Oriented 的資料庫，是屬於 NoSQL 這一邊的，經常被拿來跟 SQL 陣營的 SQLite 比較，不過我個人是覺得這沒什麼好比的，都很好用，但是 LiteDB 不用下 SQL 語法，而且它有支援到 .Net Standard 2.0，意謂著 LiteDB 是可以跨平台的，我們可以在 Windows、Linux、macOS、Android、iOS 上使用它，非常適合在 Local 端拿來儲存資料，甚至於網站在 POC 期間這種使用者人數不多的情況之下，都可以先用 LiteDB 撐著。


連線字串
跟一般的資料庫一樣，我們需要透過連線字串來建立一個 LiteDB 的連線，而 LiteDB 的連線字串選項有很多，我們只要先關注這幾個就行了。

Filename：它可以是一個檔案的絕對路徑，或是相對於與 LiteDB.dll 相同目錄的檔案路徑。
Password：如果有輸入密碼，LiteDB 就會拿密碼來為我們的資料庫檔案做 AES 加密。
Mode：有 Exclusive、ReadOnly、Shared 三個選項，這邊要注意的是預設值，在 .Net Framework 3.5、4.0、.Net Standard 2.0 預設是 Shared，但是 .Net Standard 1.3 是 Exclusive，而 Exclusive Mode 同時只能有一個 LiteDB Instance 去存取資料庫檔案。
Limit Size：資料庫檔案大小上限，我們可以這樣填 1KB、2MB、3GB。


建立 Collection
Collection 是操作 LiteDB 的基礎，資料的增刪改查都從 Collection 起手，建立 Collection 的方法有三個，我們直接看 Code。

using (var db = new LiteDatabase(connectionString))
{
    // 直接用自訂型別當 Collection Name
    var collection1 = db.GetCollection<Sample>();

    // 額外定義一個 Collection Name
    var collection2 = db.GetCollection<Sample>("customer");

    // 沒有自訂型別也可以建立 Collection
    var collection3 = db.GetCollection("customer");
}
如果建立 Collection 的時候沒有指定一個型別會怎樣？　LiteDB 會用 BsonDocument 當成 document 的型別，而這之後我們就只能以 key-value 的形式來存取 document 內的資料。


新增資料
LiteDB 提供有 Insert 及 InsertBulk 方法，新增單筆資料只能用 Insert 方法，而要新增多筆資料則 Insert 及 InsertBulk 方法都可以。

using (var db = new LiteDatabase(connectionString))
{
    var collection = db.GetCollection<Sample>();

    // 新增單筆資料
    collection.Insert(new Sample());

    // 新增多筆資料
    collection.Insert(new List<Sample>());
    collection.InsertBulk(new List<Sample>());
}
可是一樣是新增多筆資料，使用 InsertBulk 差在哪裡？　只有差在節省記憶體而已，以我實際測試寫入 150000 筆資料，每筆資料大概 100 bytes 來說，InsertBulk 反而已還比較慢，不過官方也沒有說 InsertBulk 比較快就是了。



刪除資料
LiteDB 提供 Delete 方法用來刪除資料，給 Delete 方法 id 值就可以刪除單筆資料，給條件就可以依條件刪除資料，給條件的方式有兩種，一種是我們熟悉的 Lambda Expression，一種是 LiteDB 內建 Query Methods。

using (var db = new LiteDatabase(connectionString))
{
    var collection = db.GetCollection<Sample>();

    // 刪除單筆資料
    collection.Delete(id);

    // 條件式刪除資料
    collection.Delete(x => x.Birthday > new DateTime(2000, 1, 1));
    collection.Delete(Query.GT("Birthday", new DateTime(2000, 1, 1)));
}



修改資料
LiteDB 是 Document-Oriented 的資料庫，我們要部份更新 document 中的某個欄位必須先把整個 document 撈出來，把我們想更新的欄位值修改好後再整個 document 寫回去，這就是 Document-Oriented 的特性，LiteDB 提供 Update 方法讓我們用來更新資料。

using (var db = new LiteDatabase(connectionString))
{
    var collection = db.GetCollection<Sample>();

    var doc = collection.FindById(id);

    doc.Name = "Hello World";

    // 更新單筆資料
    collection.Update(doc);
    collection.Update(id, doc);

    var docs = collection.Find(x => x.Birthday > new DateTime(2000, 1, 1)).ToList();

    foreach (var sample in docs)
    {
        sample.Birthday = new DateTime(2000, 1, 1);
    }

    // 更新多筆資料
    collection.Update(docs);
}



同場加映 Insert Or Update
通常我們做 Insert Or Update 至少就要寫三段程式碼，先到資料庫找、找到了就執行更新、找不到就執行新增，而 LiteDB 提供了一個非常方便的方法叫 Upsert，它已經幫我們把 Insert Or Update 的邏輯封裝起來了，我們直接呼叫就行了。

using (var db = new LiteDatabase(connectionString))
{
    var collection = db.GetCollection<Sample>();

    var doc = new Sample
                  {
                      Id = id,
                      Name = "Hello World",
                      Phone = "+886912345678",
                      Address = "中和",
                      Birthday = new DateTime(2000, 1, 1),
                      Code = 12345
                  };

    // Insert Or Update 單筆資料
    collection.Upsert(doc);
    collection.Upsert(id, doc);

    // Insert Or Update 多筆資料
    collection.Upsert(new List<Sample> { doc });
}



查詢資料
查詢 LiteDB 裡面的資料最基本的就是用 Find 方法，Find 方法裡面可以帶入 Lambda Expression 及 Query Method 兩種參數當做查詢條件。

using (var db = new LiteDatabase(connectionString))
{
    var collection = db.GetCollection<Sample>();

    // 給入 Lambda Expression 當條件
    var results1 = collection.Find(x => x.Name.Equals("Johnny Smith"));

    // 給入 Query Method 當條件
    var results2 = collection.Find(Query.EQ(nameof(Sample.Name), "Johnny Smith"));
}
以上針對 LiteDB 最基本的 CURD 做介紹，在我們思考著要在 Local 端儲存資料時，LiteDB 是一個很好的選項。


使用 Index 加入查詢條件
上一篇介紹 LiteDB 基本的 CRUD，但是我相信這還是有點不太夠的，隨著寫入的資料愈來愈多，做非主鍵條件查詢肯定是會愈來愈慢的，LiteDB 在沒有建 Index 的情況之下，如果查詢條件不是主鍵，它是對整個 DB 做 Full Document Scan，意謂著 LiteDB 必須將每一筆資料反序化出來之後一筆一筆去比對，不僅慢又浪費記憶體。



EnsureIndex()
在 LiteDB 建 Index 的方式非常簡單，選定欄位、呼叫 EnsureIndex() 就完成了，而且只要呼叫一次就有作用，即使重覆呼叫，已經建立的 Index 也不會重建。

using (var db = new LiteDatabase(ConnectionString))
{
    var collection = db.GetCollection<Sample>();

    collection.EnsureIndex(x => x.Name);
}
原本 15 萬筆的資料在沒有建 Index 的情況下，查詢 Name 欄位等於某個特定值的時間要 11 秒，建完 Index 只要 3 毫秒，有沒有 Index 真的差很多。



使用 Index 加入查詢條件
舉個例子來說，我們現在要找出最晚出生的人是誰？但是，程式碼千萬不要這樣寫：

using (var db = new LiteDatabase(ConnectionString))
{
    var collection = db.GetCollection<Sample>();

    var firstBorn = collection.FindAll().OrderByDescending(x => x.Birthday).First();
}
這樣寫意謂著把資料都序列化後才對 Birthday 降冪排序取第一筆，慢又消耗記憶體。



我們改用 Query Method 利用已經建好的 Index 來做查詢：

using (var db = new LiteDatabase(ConnectionString))
{
    var collection = db.GetCollection<Sample>();

    var firstBorn = collection.Find(Query.All(nameof(Sample.Birthday), Query.OrderByDescending), limit: 1).Single();
}
消耗的資源立馬就降下來了



再加個條件，要找出 2000 年以前最晚出生的是誰？經過剛剛的例子我們知道不應該這樣寫：

using (var db = new LiteDatabase(ConnectionString))
{
    var collection = db.GetCollection<Sample>();

    var firstBorn = collection.Find(x => x.Birthday < new DateTime(2000, 1, 1))
        .OrderByDescending(x => x.Birthday)
        .First();
}
應該改這樣：

using (var db = new LiteDatabase(ConnectionString))
{
    var collection = db.GetCollection<Sample>();

    var firstBorn = collection.Find(
            Query.Where(nameof(Sample.Birthday), value => value.AsDateTime < new DateTime(2000, 1, 1), Query.Descending),
            limit: 1)
        .Single();
}


限制
LiteDB 的 Index 有兩個限制：

拿來建立 Index 的欄位值必須小於 512 bytes。
每個 Collection 最多只能建 16 個 Index（包含 PrimaryKey）。




----------
20190627

LiteDB.Shell

Some common commands:
open yourdatafile.db - Open your datafile
show collections
db.your_collection_name.find - List all documents inside your collection
db.your_collection_name.ensureIndex name - Create an index in name document field
db.your_collection_name.find name = 'John' - Search for documents with name = John 
help full - Show all commands in shell. 

> open c:/temp/MyData414.db
> show collections
issues
> db.issues.find
[1]: {"_id":{"$guid":"95c6c797-97eb-4cb6-8847-0e365f4f6dab"},"DateTime":{"$date":"2019-06-27T05:10:22.8100000Z"},"ErrorText":"eee","IssueType":"Error"}
[2]: {"_id":{"$guid":"96e9c6c2-f4f7-46c6-afb7-e3209106ed00"},"DateTime":{"$date":"2019-06-27T05:09:16.0010000Z"},"ErrorText":"bbbb","IssueType":"Info"}
[3]: {"_id":{"$guid":"f69c8576-5f10-4788-8ed1-45ec32cc097a"},"DateTime":{"$date":"2019-06-27T05:08:05.8660000Z"},"ErrorText":"bbb","IssueType":"Warning"}
>


----------
20190612

Basic example
Store files
Fluent entity mapping
Stream as database
DbRef for cross references


Basic example
// Basic example
public class Customer
{
    public int Id { get; set; }
    public string Name { get; set; }
    public string[] Phones { get; set; }
    public bool IsActive { get; set; }
}

// Open database (or create if not exits)
using(var db = new LiteDatabase(@"MyData.db"))
{
    // Get customer collection
    var customers = db.GetCollection<Customer>("customers");

    // Create your new customer instance
    var customer = new Customer
    { 
        Name = "John Doe", 
        Phones = new string[] { "8000-0000", "9000-0000" }, 
        IsActive = true
    };

    // Insert new customer document (Id will be auto-incremented)
    customers.Insert(customer);

    // Update a document inside a collection
    customer.Name = "Joana Doe";

    customers.Update(customer);

    // Index document using a document property
    customers.EnsureIndex(x => x.Name);

    // Use Linq to query documents
    var results = customers.Find(x => x.Name.StartsWith("Jo"));
}


Store files
// Store files
using(var db = new LiteDatabase("MyData.db"))
{
    // Upload a file from file system
    db.FileStore.Upload("/my/file-id", @"C:\Temp\picture1.jgn");
    
    // Upload a file from Stream
    db.FileStore.Upload("/my/file-id", myStream);
    
    // Open as an stream
    var stream = db.FileStore.OpenRead("/my/file-id");
    
    // Write to another stream
    stream.CopyTo(Response.Output);
    
}


Fluent entity mapping
// Custom entity mapping to BsonDocument
            
// Re-use mapper from global instance
var mapper = BsonMapper.Global;            

mapper.Entity<Customer>()
        .Key(x => x.CustomerKey)
        .Field(x => x.Name, "customer_name")
        .Ignore(x => x.Age);
            
using(var db = new LiteDatabase(@"MyData.db"))
{
    var doc = mapper.ToDocument(new Customer { ... });
    var json = JsonSerializer.Serialize(doc, true);
    
    /* json:
    {
        "_id": 1,
        "customer_name": "John Doe"
    }
    */
}



Stream as database
// In memory database
var mem = new MemoryStream();

using(var db = new LiteDatabase(mem))
{
    ...
}

// Get database as binary array
var bytes = mem.ToArray();

// LiteDB support any Stream read/write as input
// You can implement your own IDiskService to persist data




DbRef for cross references
// DbRef to cross references
public class Customer
{
    public int Id { get; set; }
    public string Name { get; set; }
}

public class Product
{
    public int Id { get; set; }
    public string Description { get; set; }
    public double Price { get; set; }
}

// DbRef to cross references
public class Order
{
    public ObjectId Id { get; set; }
    public DateTime OrderDate { get; set; }
    public Customer Customer { get; set; }
    public List Products { get; set; }
}       

// Re-use mapper from global instance
var mapper = BsonMapper.Global;

// Produts and Customer are other collections (not embedded document)
// you can use [BsonRef("colname")] attribute
mapper.Entity<Order>()
    .DbRef(x => x.Products, "products")
    .DbRef(x => x.Customer, "customers");
            
using(var db = new LiteDatabase(@"MyData.db"))
{
    var customers = db.GetCollection("customers");
    var products = db.GetCollection("products");
    var orders = db.GetCollection("orders");

    // create examples
    var john = new Customer { Name = "John Doe" };
    var tv = new Product { Description = "TV Sony 44\"", Price = 799 };
    var iphone = new Product { Description = "iPhone X", Price = 999 };
    var order1 = new Order { OrderDate = new DateTime(2017, 1, 1), Customer = john, Products = new List() { iphone, tv } };
    var order2 = new Order { OrderDate = new DateTime(2017, 10, 1), Customer = john, Products = new List() { iphone } };

    // insert into collections
    customers.Insert(john);
    products.Insert(new Product[] { tv, iphone });
    orders.Insert(new Order[] { order1, order2 });

    // create index in OrderDate
    orders.EnsureIndex(x => x.OrderDate);

    // When query Order, includes references
    var query = orders
        .Include(x => x.Customer)
        .Include(x => x.Products)
        .Find(x => x.OrderDate == new DateTime(2017, 1, 1));

    // Each instance of Order will load Customer/Products references
    foreach(var c in query)
    {
        Console.WriteLine("#{0} - {1}", c.Id, c.Customer.Name);

        foreach(var p in c.Products)
        {
            Console.WriteLine(" > {0} - {1:c}", p.Description, p.Price);
        }
    }
}

