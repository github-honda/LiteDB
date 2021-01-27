From: hondachen@hotmail.com
Date: 2021-01-17
Subject: readme-LiteDB.txt

本文網址: https://github.com/github-honda/LiteDB/blob/master/doc/readme-LiteDB.txt
歡迎來信交流.

LiteDB: An open source MongoDB-like database with zero conﬁguration - mobile ready

參考:
https://github.com/github-honda/LiteDB
https://github.com/mbdavid/LiteDB
https://github.com/mbdavid/LiteDB/wiki
http://www.litedb.org/docs/ v5 文件
http://www.litedb.org/
  

摘要:
1. LiteDB 的控制細節, 若太久沒用, 很容易忘記規則, 造成混淆, 不好維護.
   LiteDB 畢竟不如 MongoDB 使用人數多, 文件說明有限, 測試案例也較少. 
   較複雜的功能, 需要 Try-Run後確認, 才能以正確的方式實作出功能.
   所以有些功能雖然很強, 建議只使用簡單的功能, 容易記憶及維護就好.
   
   
2. 盡量使用 Index 查詢才能加速查詢速度. 
   使用 Find() 搭配 EnsureIndex():
     先 Find查詢後, 再交給 LINQ 處理.
     col.EnsureIndex(x => x.Name);
     var result = col
       .Find(Query.EQ("Name", "John Doe")) // This filter is executed in LiteDB using index
       .Where(x => x.CreationDate >= x.DueDate.AddDays(-5)) // This filter is executed by LINQ to Object
       .OrderBy(x => x.Age)
       .Select(x => new 
       { 
           FullName = x.FirstName + " " + x.LastName, 
           DueDays = x.DueDate - x.CreationDate 
       }); // Transform

   查詢時指定 index
     查詢(指定 index 的前 iCount 筆資料, "_id" 為PK.)
       int iOrder = bAscending ? Query.Ascending : Query.Descending;
       col.Find(Query.All("_id", iOrder), 0, iCount);
       col.Find(Query.All("FType", iOrder), 0, iCount);

     查詢最後100筆資料:	
       Create an index on AddedTime
       Run `collection.Find(Query.All("AddedTime", Query.Descending), 0, 100);
       Now you will list all yor documents in AddedTime desc order and get only 100 first.

   建立索引, 名稱為"Key", 由兩個欄位組成一個字串建立.
     col.EnsureIndex("Key", "$.DebtorNumber + '_' + $.ProductSku", true);

   多欄位索引:
     // 建立索引名稱為"my_index", 由兩個欄位組成 BsonArray.
     col.EnsureIndex("my_index", "[$.Name, $.OrderNumber]", false);. 
     // 查詢時指定 Index 為 "my_index":  
     col.Find(Query.EQ("my_index", new BsonArray("John", 123)).

   Samples of EnsureIndex:
     col.EnsureIndex("idx_name", "name", false); 盡量用這個方式比較明確!
     col.EnsureIndex("idx_name", x => x.name, false); 同上
	 
     col.EnsureIndex("attr1", "$.attr1");
     col.EnsureIndex("LowerName", "LOWER($.Name)");

	 // no index name defined
     col.EnsureIndex("$.name");
     col.EnsureIndex(x => x.name, false); 同上

	 col.EnsureIndex("name.last");
	 col.EnsureIndex("$.name.first", true);

     // 建立索引名稱為"idx_name", 由兩個欄位組成 BsonArray.
     col.EnsureIndex("idx_name", "[$.Name, $.OrderNumber]", false);. 
     // 查詢時指定 Index 為 "idx_name":  
     col.Find(Query.EQ("idx_name", new BsonArray("John", 123)).
	 

3. 開啟資料庫後就以 EnsureIndex 建立index, 集中一個地方呼叫 EnsureIndex 就可以!

4. 沒有建立 Index的欄位, 不能用 Find(), 只能改用 Query() full scan collection.
        public IEnumerable<DKeyType> QueryAllOrderBySeqNo()
        {
            // 沒有建立 Index的欄位, 不能用 Find(), 只能改用 Query() full scan collection.
            return GetCollection()
                .Query()
                // .Where(x => x.FType.StartsWith("any"))
                .OrderBy(x => x.FSeqNo, Query.Ascending)
                .ToEnumerable();
        }
		

----------
原文摘要 
Document
  將 Document class 獨立出來, 比較容易維護: 雖然可以把 Document 跟 Collection 合併在一個 Class 中, 但是卻容易造成(LiteDB 跟 AP)之間的功能混淆.

Data Structure
  BSON (Binary JSON)
    提供 JsonSerializer 功能實作 JSON
	BsonMapper 可將 Class 物件轉為 BsonValue, 例如:
	  var customer = new Customer { Id = 1, Name = "John Doe" };
      var doc = BsonMapper.Global.ToDocument(customer);
      var jsonString = JsonSerialize.Serialize(doc);
	
	JsonSerialize 還支援讀寫到 file 或 stream.
    JsonSerialize also supports TextReader and TextWriter to read/write directly from a file or Stream
	
  ObjectId
    ObjectId is a 12 bytes BSON type, 依序編號, 適合當索引欄位.
      Timestamp: 內建(UTC 建立時間) Value representing the seconds since the Unix epoch (4 bytes)
      Machine: Machine identifier (3 bytes)
      Pid: Process id (2 bytes)
      Increment: A counter, starting with a random value (3 bytes)	
    JSON representation: { "$oid": "507f1f55bcf96cd799438110" } 12 bytes in hex format
	依序編號. ObjectIds are sequential, 適合當索引欄位.
      var id = ObjectId.NewObjectId();

      // You can get creation datetime from an ObjectId
      var date = id.CreationTime;

      // ObjectId is represented in hex value
      Debug.WriteLine(id);
      "507h096e210a18719ea877a2"

      // Create an instance based on hex representation
      var nid = new ObjectId("507h096e210a18719ea877a2");	
  
  
Object Mapping 物件轉換對應
  The LiteDB mapper converts POCO classes documents. When you get a ILiteCollection<T> instance from LiteDatabase.GetCollection<T>, T will be your document type. If T is not a BsonDocument, LiteDB internally maps your class to BsonDocument. To do this, LiteDB uses the BsonMapper class:
    當呼叫 LiteDatabase.GetCollection<T> 時, 若 傳入的 T (原本的class物件), 不是 BsonDocument時, 則會把 T 轉為 BsonDocument.
    POCO = Plain Old CLR Objects is a class, which doesn't depend on any framework-specific base class.
  Mapper Conventions 欄位轉換原則
    欄位為(僅唯讀 或 可讀寫)
  
    _id欄位
	  至少要有一個被視為 _id 的欄位:
        欄名為 (Id 不分大小寫),
        或 欄名為<ClassName>Id, 
        或 標示[BsonId] Attribute,  
        或 以 fluent API (x => x.欄位) 
      型別可以為(ObjectID, Guid, Int32, Int64, string)
      不可存放 value = Null, MinValue or MaxValue.
	  會自動建立索引 unique index. 因此最大為 256 bytes. 

    AutoId欄位 
      若要自動編號, 則Insert時, (不要傳入 AutoId 的欄位) 或 (將 AutoId 的欄位清為 empty). 
	    empty 例如: Int 型別為0, Guid型別為 Guid.Empty.
		
	  會自動編號的型別
        ObjectId: ObjectId.NewObjectId()
        Guid: Guid.NewGuid() method
        Int32/Int64: New collection sequence. 從1開始編號. 並以目前資料中(最大號+1)編號.	    
	  (AutoId 欄位)必須可讀寫. AutoId requires the id field to have a public setter.
	  (AutoId 欄位)在Insert時, 執行自動編號的規則為:
	    BsonDocument:            若沒有 _id 欄位, 就會執行自動編號. 
		strongly-typed document: 若(_id 欄位為 empty), 則先(刪除_id欄位)後, 再Insert. (empty 例如: Int 型別為0, Guid型別為 Guid.Empty)
          AutoId is only used when there is no _id field in the document upon insertion. 
		  In strongly-typed documents, BsonMapper removes the _id field for empty values (like 0 for Int or Guid.Empty for Guid). 
		  Please note that AutoId requires the id field to have a public setter.
		  
	  
	忽略欄位: 標示 [BsonIgnore] 的欄位, LiteDB 會忽略不處理.
    指定欄名: 標示 [BsonField("fieldName")]可指定欄名.
	
Collections
  Collection 存放 Documents. 類似 RDB 的 Table 存放 Rows.
  Collection 名稱不分大小寫.
    Collection 名稱字頭為 "_"者, 保留為內部儲存用途.
    Collection 名稱字頭為 "$"者, 保留為內部系統/虛擬的 Collections.
  當執行到第一個 Insert()或 EnsureIndex()時, 會自動建立 Collection.
  以下兩個範例產生相同的結果:
    // Typed collection
    using(var db = new LiteDatabase("mydb.db"))
    {
      // Get collection instance
      var col = db.GetCollection<Customer>("customer");
    
      // Insert document to collection - if collection does not exist, it is created
      col.Insert(new Customer { Id = 1, Name = "John Doe" });
    
      // Create an index over the Field name (if it doesn't exist)
      col.EnsureIndex(x => x.Name);
    
      // Now, search for your document
      var customer = col.FindOne(x => x.Name == "john doe");
    }

    // Untyped collection (T is BsonDocument)
    using(var db = new LiteDatabase("mydb.db"))
    {
      // Get collection instance
      var col = db.GetCollection("customer");
    
      // Insert document to collection - if collection does not exist, it is created
      col.Insert(new BsonDocument{ ["_id"] = 1, ["Name"] = "John Doe" });
    
      // Create an index over the Field name (if it doesn't exist)
      col.EnsureIndex("Name");
    
      // Now, search for your document
      var customer = col.FindOne("$.Name = 'john doe'");
    }  

  System Collections
    以 Collections 的型態提供資料庫資訊.
    System Collections 都以 "$" 開頭.
      除了 $file 之外, 都是 read-only.	


Collection	        Description
------------------  -----------------------------
$cols	            Lists all collections in the datafile, including the system collections.
$database	        Shows general info about the datafile.
$indexes	        Lists all indexes in the datafile.
$sequences	        Lists all the sequences in the datafile.
$transactions	    Lists all the open transactions in the datafile.
$snapshots	        Lists all existing snapshots.
$open_cursors	    Lists all the open cursors in the datafile.
$dump(pageID)	    Lists advanced info about the desired page. If no pageID is provided, lists all the pages.
$page_list(pageID)	Lists basic info about the desired page. If no pageID is provided, lists all the pages.
$query(subquery)	Takes a query as string and returns the result of the query. Can be used for simulating subqueries. Experimental.
------------------  -----------------------------
使用方式:
        /// <summary>
        /// 回傳所有的 collection 名稱.
        /// 包括(user 跟 system).
        /// </summary>
        /// <returns></returns>
        public ILiteCollection<BsonDocument> ZSysCols()
        {
            return Open().GetCollection("$cols");
        }

例如 LiteDatabase.GetCollectionNames() 內部就是以 "$cols" 製作如下:
        public IEnumerable<string> GetCollectionNames()
        {
            // use $cols system collection with type = user only
            var cols = this.GetCollection("$cols")
                .Query()
                .Where("type = 'user'")
                .ToDocuments()
                .Select(x => x["name"].AsString)
                .ToArray();

            return cols;
        }
------------------  -----------------------------
$file(path)	See below.	  
$file
The $file system collection can be used to read from and write to external files.

SELECT $ INTO $FILE('customers.json') FROM Customers dumps the entire content from the collection Customers into a JSON file.
SELECT $ FROM $FILE('customers.json') reads the entire content from the JSON file.
There is also limited support for CSV files. Only basic data types are supported and, on writing, the schema of the first document returned by the query will define the schema of the entire CSV file, with additional fields being ignored.

SELECT $ INTO $FILE('customers.csv') FROM Customers dumps the entire content from the collection Customers into a CSV file.
SELECT $ FROM $FILE('customers.csv') reads the entire content from the CSV file.
The single parameter can be:

$file("filename.json|csv"): Support filename only with file extension as format
$file({ options }): Support all options
{
    filename: "string",
    format: "json|csv",
    enconding: "utf-8",
    overwritten: false,
    // JSON only
    ident: 4,
    pretty: false
    // CSV only
    delimiter: ",",
    header: true, // write CSV header columns
    header: [] // define header columns names when read
}


BsonDocument
  以 Dictionary<string, BsonValue> 儲存 Key-Value 型態的資料.
  範例:
    1. 建立方式1: 簡化設定 var doc = new BsonDocument { ["_id"] = 10, ["name"] = "John" };
	2. 建立方式2: 設定 BsonDocument[Key].Value
	        var customer = new BsonDocument();
            customer["_id"] = ObjectId.NewObjectId();
            customer["Name"] = "John Doe";
            customer["CreateDate"] = DateTime.Now;
            customer["Phones"] = new BsonArray { "8000-0000", "9000-000" };
            customer["IsActive"] = true;
            customer["IsAdmin"] = new BsonValue(true);
            customer["Address"] = new BsonDocument
            {
                ["Street"] = "Av. Protasio Alves"
            };

            customer["Address"]["Number"] = "1331";

            var json = JsonSerializer.Serialize(customer);
			
	3. 讀取 [Key].Value
        public IEnumerable<string> GetCollectionNames()
        {
            // 取得 ILiteCollection<BsonDocument>:
            var cols = this.GetCollection("$cols")
                .Query()
                .Where("type = 'user'")
                .ToDocuments()
                .Select(x => x["name"].AsString)
                .ToArray();

            return cols;
        }

    4. BsonDocuments 轉為 KeyValuePair<string, BsonValue>[]	
	public void Document_Copies_Properties_To_KeyValue_Array()
	{
		// ARRANGE
		// create a Bson document with all possible value types
		var document = new BsonDocument();
		document.Add("string", new BsonValue("string"));
		document.Add("bool", new BsonValue(true));
		document.Add("objectId", new BsonValue(ObjectId.NewObjectId()));
		document.Add("DateTime", new BsonValue(DateTime.Now));
		document.Add("decimal", new BsonValue((decimal) 1));
		document.Add("double", new BsonValue((double) 1.0));
		document.Add("guid", new BsonValue(Guid.NewGuid()));
		document.Add("int", new BsonValue((int) 1));
		document.Add("long", new BsonValue((long) 1));
		document.Add("bytes", new BsonValue(new byte[] {(byte) 1}));
		document.Add("bsonDocument", new BsonDocument());

		// ACT
		// copy all properties to destination array
		var result = new KeyValuePair<string, BsonValue>[document.Count()];
		document.CopyTo(result, 0);
	}
		
    
  Keys
    不分大小寫
	不可重複
	除了 _id 欄位會排在第一個位置以外, 其餘的欄位會依照原本的順序存放.
  Values
    Value 可存放 Null, Int32, Int64, Decimal, Double, String, Embedded Document, Array, Binary, ObjectId, Guid, Boolean, DateTime, MinValue, MaxValue
    作為索引的欄位, 在 BSON serialization 後, 最大為 256 bytes.  
	.NET 擴充的 Value型別:  BsonValue, BsonArray, BsonDocument
      BsonValue
	    可存放 any BSON data type, including null, array or document.
		內建所有 .NET 資料型別建構式.
		唯讀不可變更. immutable.
		屬性 RawValue 可回傳內部實際的物件實體(.NET object instance)
		
		已實作隱含轉換 public static implicit operator, 所以不需要以下明確轉換: 
			.NET types to BsonValue, implicit 
			  new BsonValue("string");
			  new BsonValue(true);
			  new BsonValue(ObjectId.NewObjectId());
			  new BsonValue(DateTime.Now);
			  new BsonValue((decimal) 1);
			  new BsonValue((double) 1.0);
			  new BsonValue(Guid.NewGuid());
			  new BsonValue((int) 1);
			  new BsonValue((long) 1);
			  new BsonValue(new byte[] {(byte) 1});
			  new BsonDocument());
			BsonValue to .NET types:
				doc["_id"].AsInt32;
				doc["_id"].AsInt64;
				doc["FirstString"].AsString;
				doc["Date"].AsDateTime;
				doc["CustomerId"].AsGuid;
				doc["EmptyString"].AsString;
				doc["maxDate"].AsDateTime;
				doc["minDate"].AsDateTime;
				doc["Items"].AsArray;
				doc["Items"].AsArray[0];
				doc["Items"].AsArray[4];
          		
	  BsonArray
	    可支援 IEnumerable<BsonValue>
		每個元素可以是不同型別的 BSON type objects.
	  BsonDocument
	    沒有的欄位會回傳 BsonValue.Null.

 
Index 索引
  ID欄位會自動建立索引.
  查詢前需要呼叫 EnsureIndex(), 才會使用到指定的Index加速查詢.
    EnsureIndex() 回傳 true=建立新索引, false=原索引已存在, 沒影響.
  作為索引的欄位, 在 BSON serialization 後, 最大為 256 bytes.  

Expressions 運算式
  $ 代表 root document.  預設可省略. 因此 $.Address.Street 等於 Address.Street.

  $ 跟 * 的差別
    $ represents current root document. When $ is used, you are referencing the root document. If neither $ nor * are present, $ is assumed.
    * represent a group of documents. Used when GROUP BY is present or when you want to return a single value in a query (SELECT COUNT(*) FROM customers).
    SELECT $ FROM customers 
	  returns IEnumerable<BsonDocument> result (N documents). 
    SELECT * FROM customers 
	  returns a single value, a BsonArray with all documents result inside.

DbRef
  文件少, 缺測試案例, 實作功能待測試.
  難維護及記憶整個控制循環的每一個細節, 放棄使用 !
  
  
ConnectionString

  ConnectionString 中, 若沒有"="的話, 則當作檔案名稱.
    檔案名稱可為絕對路徑, 或(相對於.dll目錄的)相對路徑. Full path or relative path from DLL directory.
	
    可選擇:
	Key			Type			Description																		Default value
	----------- --------------- ------------------------------------------------------------------------------- ---------
	Filename	string			Full or relative path to the datafile. 
	                            Supports :memory: for memory database or :temp: for in disk temporary database 
								(file will deleted when database is closed) [required]
	Connection	string			Connection type (“direct” or “shared”)											"direct”
	Password	string			Encrypt (using AES) your datafile with a password								null (no encryption)
	InitialSize	string or long	Initial size for the datafile (string suppoorts “KB”, “MB” and “GB”)			0
	ReadOnly	bool			Open datafile in read-only mode													false
	Upgrade		bool			Check if datafile is of an older version and upgrade it before opening			false
	----------- --------------- ------------------------------------------------------------------------------- ---------
	
  Connection Type
    LiteDB offers 2 types of connections: Direct and Shared. This affect how engine will open data file.
    Direct: Engine will open the datafile in exclusive mode and will keep it open until Dispose(). 
	        The datafile cannot be opened by another process. 
			This is the recommended mode because it’s faster and cachable.
			預設為(Direct互斥模式), 不可同時開啟, 直到關閉檔案為止. 建議使用這種模式, 速度快且有快取.
    Shared: Engine will be close the datafile after each operation. 
	        Locks are made using Mutex. 
			This is more expensive but you can open same file from multiple processes.	
			(Shared共用模式)則是每次作業後關閉檔案. 以 Mutex 方式鎖定檔案. 
			系統負荷較重, 但是不同作業可以開啟相同的檔案.
  Examples:
    App.config
    <connectionStrings>
        <add name="LiteDB" connectionString="Filename=C:\database.db;Password=1234" />
    </connectionStrings>  
  
    C#
    System.Configuration.ConfigurationManager.ConnectionStrings["LiteDB"].ConnectionString

  
            // only filename
            var onlyfile = new ConnectionString(@"demo.db");

            // file with spaces without "
            var normal = new ConnectionString(@"filename=c:\only file\demo.db");

            // file with spaces with " and ;
            var full = new ConnectionString(
                @"filename=""c:\only;file\""d\""emo.db""; 
                  password =   ""john-doe "" ;
                  initial size = 10 MB ;
                  readONLY =  TRUE;");
				  
        /// <summary>
        /// Full path or relative path from DLL directory. Can use ':temp:' for temp database or ':memory:' for in-memory database. (default: null)
        /// </summary>
        public string Filename { get; set; }
		
            // Can use ':temp:' for temp database or ':memory:' for in-memory database. (default: null)
            using (var db = new LiteDatabase(":temp:"))
            using (var db = new LiteDatabase(":memory:"))
	
				  

            // ConnectionString_Very_Long	
			Filename length = 49
			Password length = 512
            var cn = new ConnectionString(@"Filename=C:\Users\yup\AppData\Roaming\corex\storecore.file;Password='1495c305c5312dd1a9a18d9502daa0369216763ca7a6f537ddbe290241cf8aad1ca326313adec74bb98d1955747347cf0e3f087899d8bb2e0aa002ff825e1c0f25eaa79e5dfbf1c0e2daf6746a3a3f140244b764204c20c0ccede3521eaf8537ae32d4b13a04f1c387f56a8d6fa095bc53451c1892a46b8182afd94559cd7377aebc8d4a2b4883c637a359e6e67e1d8c2d789721351ebb000409329b2e875d21278b7c76724c68729e53dac50168564b8c3432018212a111c952e593829b42c296458cc0020174aaef9ca6b5661ca965004404c2bbb256bc41a8aa5c5349c615e40328a3263c45e5f96e61048149e98aa8b6f2afb59d73379e1dce5429752d8d'");

			connection=shared:			
			class Program
			{
				static async Task Main(string[] args)
				{
					await Task.WhenAll(
						Action(),
						Action());
				}
				private static async Task Action()
				{
					using var db = new LiteDatabase(@"Filename=database.db;Password='1234';connection=shared");
					await Task.Delay(1000);
				}
			}			
			If you prefer using a connectionString object instead of a literal string, you can also replace the string with something like this

			new ConnectionString(@"database.db")
			{
				Password = "1234",
				Connection = ConnectionType.Shared
			};  

MultiKey Index
  LiteDB 的 MultiKey Index 是指(以BsonArray 建立的欄位可包含多欄位), 不是傳統的(兩個或多個欄位做索引).

[BsonCtor]
  Starting with version 5 of LiteDB you can use BsonCtorAttribute to indicate which constructor the mapper must use. Fields no longer need to have a public setter and can be initialized by the constructor.
  標示 [BsonCtor]屬性可指定 constructor 給 LiteDB mapper 使用
  GetCollection<T> 會依照以下順序取用 Constructor:
    1. 先找有(標示 [BsonCtor] 的 Constructor).
	2. 否則找(沒有參數的 Constructor).
	3. 最後找(參數名稱符合欄位名稱的 Constructor)
	
Pragmas
  In LiteDB v5, pragmas are variables that can alter the behavior of a datafile. They are stored in the header of the datafile.
  Rebuild Options
    Rebuild 可重整資料庫, 縮小並加速處理速度.
      rebuild; rebuilds the database with the default collation and no password
      rebuild {"collation": "en-GB/IgnoreCase"}; rebuilds the datafile with the en-GB culture and case-insensitive string comparison
      rebuild {"collation": "pt-BR/None", "password" : "1234"}; rebuilds the datafile with the pt-BR culture, case-sensitive string comparison and sets the password to “1234”	


----------
20210113

檔案不可同時使用:
System.IO.IOException
  HResult=0x80070020
  Message=由於另一個處理序正在使用檔案 'W:\Research\ZLib46\ZLib\ZLib46200101\ZLib\CRUDLiteDBWinForm\bin\Debug\MyTest1.db'，所以無法存取該檔案。
  Source=mscorlib
  StackTrace:
   at System.IO.__Error.WinIOError(Int32 errorCode, String maybeFullPath)
   at System.IO.FileStream.Init(String path, FileMode mode, FileAccess access, Int32 rights, Boolean useRights, FileShare share, Int32 bufferSize, FileOptions options, SECURITY_ATTRIBUTES secAttrs, String msgPath, Boolean bFromProxy, Boolean useLongPath, Boolean checkHost)
   at System.IO.FileStream..ctor(String path, FileMode mode, FileAccess access, FileShare share, Int32 bufferSize, FileOptions options)
   at LiteDB.Engine.FileStreamFactory.GetStream(Boolean canWrite, Boolean sequencial) in W:\Research\ZLib46\ZLib\ZLib46200101\ZLib\LiteDB\Engine\Disk\StreamFactory\FileStreamFactory.cs:line 43
   at LiteDB.Engine.StreamPool.<>c__DisplayClass3_0.<.ctor>b__0() in W:\Research\ZLib46\ZLib\ZLib46200101\ZLib\LiteDB\Engine\Disk\StreamFactory\StreamPool.cs:line 29
   at System.Lazy`1.CreateValue()


        /// <summary>
        /// Create new data file FileStream instance based on filename
        /// </summary>
        public Stream GetStream(bool canWrite, bool sequencial)
        {
            var write = canWrite && (_readonly == false);

            var isNewFile = write && this.Exists() == false;

            var stream = new FileStream(_filename,
                _readonly ? System.IO.FileMode.Open : System.IO.FileMode.OpenOrCreate,
                write ? FileAccess.ReadWrite : FileAccess.Read,
                write ? FileShare.Read : FileShare.ReadWrite,
                PAGE_SIZE,
                sequencial ? FileOptions.SequentialScan : FileOptions.RandomAccess);

            if (isNewFile && _hidden)
            {
                File.SetAttributes(_filename, FileAttributes.Hidden);
            }

            return _password == null ? (Stream)stream : new AesStream(_password, stream);
        }


----------
20210102
多欄位 Index

https://gitter.im/mbdavid/LiteDB?at=5c00f2521c439034aff5d55f

Yes, but not a physical new property, a new index. 
LiteDB has no support for compose indexes, but you can create complex indexes. 
You can create something like this: 
  col.EnsureIndex("my_index", "[$.Name, $.OrderNumber]", false);. 

This will create a new index in collection that each key contains an array with 2 elements: Name, OrderNumber. 
To use this in query, you can: 
  col.Find(Query.EQ("my_index", new BsonArray("John", 123)).
  
  
// 建立 "my_index" 為 Name + OrderNumber 兩個欄位
// 查詢時指定 Index 為 "my_index":  
  col.EnsureIndex("my_index", "[$.Name, $.OrderNumber]", false);. 
  col.Find(Query.EQ("my_index", new BsonArray("John", 123)).

// Query.All 代表指定 index 為 "_id" 欄位.
  col.Find(Query.All(iOrder), 0, iCount);

----------
20201231
查詢 Index 的最後100筆資料

https://github.com/mbdavid/LiteDB/issues/37
If you want take your lasted 100 docs using indexes, you can:

Create an index on AddedTime
Run `collection.Find(Query.All("AddedTime", Query.Descending), 0, 100);
Now you will list all yor documents in AddedTime desc order and get only 100 first.
  
----------
20201231
使用 (EnsureIndex + Find) 比 Select-Where 快:
https://github.com/mbdavid/LiteDB/issues/969

Hi @LennardF1989, thanks for share this. When we talk about query performance, the most important think is: has a good index to be used! Good indexes are based in how distinct keys they have. If this combination of debtorNumber + productSkus are unique: perfect. Create an index on this and use Query.In, like this:

public IEnumerable<GetPriceResponse> Get(int debtorNumber, List<string> productSkus)
{
    var keys = productSkus.Select(x => debtorNumber + "_" + x)
        .Select(x => new BsonValue(x))
        .ToList();
    
    // prefer use this "EnsureIndex" in you database open (to avoid checking this index in all query)
    // if you do not have this "Key" field in your class, you can create an expression:
    // I'm create as unique (set false if not possible)
    _liteCollection.EnsureIndex("Key", "$.DebtorNumber + '_' + $.ProductSku", true);
    
    // now, use Query.In
    var result = _liteCollection.Find(Query.In("Key", keys));

    return result.Select(x => x.ServiceResponse);
}
In v4, this should be fastest way to get this data. Check this and let me know how was the result.

  
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

