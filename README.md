# ORM
## SQLite数据库框架

版本：2.3.5<br>
作者：西门提督<br>
日期：2016-12-15

## ORM数据库框架用法如下：

### 1.AndroidManifest.xml添加权限：
    <!-- 往sdcard中写入数据的权限 -->
    <uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE" />
    <!-- 在sdcard中创建/删除文件的权限 -->
    <uses-permission android:name="android.permission.MOUNT_UNMOUNT_FILESYSTEMS" />
    <!-- 读取sdcard权限 -->
    <uses-permission android:name="android.permission.READ_EXTERNAL_STORAGE" />

### 2.BaseApplication初始化操作：
        public class BaseApplication extends Application {

            public static BaseApplication app;
            private DbHelper.DaoConfig daoConfig;

            @Override
            public void onCreate() {
                super.onCreate();
                app = this;

                daoConfig = new DbHelper.DaoConfig()
                        .setDbName("simon.db") // 设置数据库名，默认simon.db
                        .setDbDir(new File(Cons.DB_DIR)) // 设置数据库路径，默认安装程序路径下
                        .setTableCreateListener(new DbHelper.TableCreateListener() { // 设置表创建的监听
                            @Override
                            public void onTableCreated(DbHelper db, TableEntity table){
                                Log.e(" >>> ", "onTableCreated：" + table.getName());
                            }
                        })
                        .setAllowTransaction(true) // 设置是否允许事务，默认true
                        .setDbVersion(2) // 设置数据库的版本号
                        .setDbOpenListener(new DbHelper.DbOpenListener() { // 设置数据库打开的监听
                            @Override
                            public void onDbOpened(DbHelper db) {
                                // 开启数据库支持多线程操作提升性能，对写入加速提升巨大
                                db.getDatabase().enableWriteAheadLogging();
                            }
                        })
                        .setDbUpgradeListener(new DbHelper.DbUpgradeListener() { // 设置数据库更新操作的监听
                            @Override
                            public void onUpgrade(DbHelper db, int oldVersion, int newVersion) {
                                // db.addColumn(Child.class, "newColumn");
                                // db.dropTable(...);
                                // ...
                                // or
                                // db.dropDb();
                            }
                        });
            }

            public DbHelper.DaoConfig getDaoConfig() {
                return daoConfig;
            }

            public static BaseApplication getInstance() {
                return app;
            }
        }

### 3.实体类示例：
        /**
         * onCreated = "sql"：当第一次创建表需要插入数据时候在此写sql语句
         * onCreated = "CREATE UNIQUE INDEX index_name ON child(name, email)"
         * name = "id"：数据库表中的一个字段
         * isId = true：是否是主键
         * autoGen = true：是否自动增长
         * property = "NOT NULL"：添加约束
         */
        @Table(name = "child")
        public class Child {

            @Column(name = "id", isId = true)
            private int id;

            @Column(name = "name")
            private String name;

            @Column(name = "email")
            private String email;

            @Column(name = "parentId" /*, property = "UNIQUE"//如果是一对一加上唯一约束*/)
            private long parentId; // 外键表id

            // 这个属性被忽略，不存入数据库
            private String willIgnore;

            @Column(name = "text")
            private String text;

            public Parent getParent(DbHelper db) throws DbException {
                return db.findById(Parent.class, parentId);
            }

            public long getParentId() {
                return parentId;
            }

            public void setParentId(long parentId) {
                this.parentId = parentId;
            }

            public int getId() {
                return id;
            }

            public void setId(int id) {
                this.id = id;
            }

            public String getName() {
                return name;
            }

            public void setName(String name) {
                this.name = name;
            }

            public String getEmail() {
                return email;
            }

            public void setEmail(String email) {
                this.email = email;
            }

            public String getWillIgnore() {
                return willIgnore;
            }

            public void setWillIgnore(String willIgnore) {
                this.willIgnore = willIgnore;
            }

            public String getText() {
                return text;
            }

            public void setText(String text) {
                this.text = text;
            }

            @Override
            public String toString() {
                return "Child{" +
                        "id=" + id +
                        ", name='" + name + '\'' +
                        ", email='" + email + '\'' +
                        ", parentId=" + parentId +
                        ", willIgnore='" + willIgnore + '\'' +
                        ", text='" + text + '\'' +
                        '}';
            }
        }

### MainActivity示例：
        public class MainActivity extends AppCompatActivity {

            private TextView tv;
            protected DbHelper db;
            private String temp;

            @Override
            protected void onCreate(Bundle savedInstanceState) {
                super.onCreate(savedInstanceState);
                setContentView(R.layout.activity_main);

                db = ORMManager.getDb(this, BaseApplication.getInstance().getDaoConfig());
                tv = (TextView) findViewById(R.id.tv);
            }

            // 用集合向Child表中插入多条数据
            public void save(View btn) throws DbException {
                List<Child> childList = DatasUtil.getChildList();
                // db.save()方法不仅可以插入单个对象，还能插入集合
                db.save(childList);
                Child child = DatasUtil.getChild();
                db.saveBindingId(child); // 保存对象关联数据库生成的id
                tv.setText(child.toString()); // 保存的同时，对象id已经赋值
            }

            // 用集合向Parent表中插入1000条数据
            public void saveList(View btn) {
                ORMManager.task().run(new Runnable() { // UI异步执行
                    @Override
                    public void run() {
                        List<Parent> parentList = DatasUtil.getParentList();
                        try {
                            long start = System.currentTimeMillis();
                            db.save(parentList);
                            temp = "批量插入1000条数据:" + (System.currentTimeMillis() - start) + "ms\n";
                        } catch (DbException e) {
                            e.printStackTrace();
                            temp = "批量插入异常";
                        }
                        final String finalResult = temp;
                        ORMManager.task().post(new Runnable() { // UI同步执行
                            @Override
                            public void run() {
                                tv.setText(finalResult);
                            }
                        });
                    }
                });
            }

            // 删除表中的数据
            public void deleteDatas(View btn) throws DbException {
                db.delete(Child.class); // Child表中数据将被全部删除
                db.delete(Parent.class); // Parent表中数据将被全部删除
                temp = "Child表与Parent表中数据全部删除";
                tv.setText(temp);
            }

            // 条件删除
            public void deleteWhere(View btn) throws DbException {
                // WhereBuilder.b().and("id",">","5").or("id","=","1").expr(" and mobile > '2015-12-29 00:00:01' ");
                db.delete(Child.class,
                        WhereBuilder.b("id", "=", 1).and("isAdmin", "=", true));
                tv.setText("条件删除成功");
            }

            // 查询删除
            public void deleteQuery(View btn) throws DbException {
                List<Parent> parentList = db.selector(Parent.class).orderBy("id", true).limit(1000).findAll();
                db.delete(parentList);
                tv.setText("查询删除成功");
            }

            // 删除数据库
            public void dropDB(View btn) throws DbException {
                db.dropDb();
                tv.setText("删除数据库成功");
            }

            // 删除表
            public void dropTable(View btn) throws DbException {
                db.dropTable(Child.class);
                db.dropTable(Parent.class);
                tv.setText("删除表成功");
            }

            // 查询Child表，将所有查询结果的email字段值修改
            public void updateFind(View btn) throws DbException {
                Child first = db.findFirst(Child.class);
                first.setEmail("updateFind@qq.com");
                db.update(first, "email"); // email：表中的字段名
                tv.setText("查询修改成功");
            }

            // 条件修改
            public void updateWhere(View btn) throws DbException {
                db.update(Parent.class,
                        WhereBuilder.b("id", "=", 1).and("isAdmin", "=", true),
                        new KeyValue("name", "test_name"), new KeyValue("isAdmin", false));
                tv.setText("条件修改成功");
            }

            // 修改对象
            public void updateObj(View btn) throws DbException {
                Child first = db.findFirst(Child.class);
                first.setEmail("simon@qq.com");
                db.saveOrUpdate(first);
                tv.setText("修改对象成功");
            }

            // 条件查询
            public void queryWhere(View btn) throws DbException {
                // 查询数据库表中第一条数据
                Child first = db.findFirst(Child.class);
                if (first != null) temp = first.toString() + "\n\n";

                // 查询出id为1，3，6中某个实体类数据
                Child test = db.selector(Child.class).where("id", "in", new int[]{1, 3, 6}).findFirst();
                if (test != null) temp += test.toString() + "\n\n";

                // 查询出id在1和5之间的某个实体
                Child test2 = db.selector(Child.class).where("id", "between", new String[]{"1", "5"}).findFirst();
                if (test2 != null) temp += test2.toString() + "\n\n";

                Child test3 = db.selector(Child.class)
                        .where("name", "=", "simon")
                        .and("age", "=", 30)
                        .findFirst();
                if (test3 != null) temp += test3.toString() + "\n\n";

                DbModel model = db.findDbModelFirst(new SqlInfo("select name from child"));
                temp += model.getString("name") + "\n\n";

                List<DbModel> persons = db.findDbModelAll(new SqlInfo("select name from child where id > 2"));
                for(DbModel person: persons){
                    temp += person.getString("name") + "\n";
                }

                // 添加查询条件进行查询
                List<Child> all = db.selector(Child.class).where(WhereBuilder.b("name", "=", "张三").and("email", "=", "zhangsan@163.com")).findAll();
                if (all != null && all.size() != 0) {
                    for (Child childInfo : all) {
                        temp += childInfo.toString() + "\n";
                    }
                }
                tv.setText(temp);
            }

            // 查询Count
            public void queryCount(View btn) throws DbException {
                // 查询出id为1，3，6的实体数据条数
                long count = db.selector(Parent.class).where("id", "in", new int[]{1, 3, 6}).count();
                tv.setText(String.valueOf(count));
            }

            // 集合查询
            public void queryList(View btn) throws DbException {
                // 查询某张表或某个实体数据集合
                List<Child> children = db.selector(Child.class).findAll();
                temp = "List<Child> Size:" + children.size() + "\n";
                tv.setText(temp);
                if (children.size() > 0) {
                    temp += "Last Child Object:" + children.get(children.size() - 1) + "\n";
                    tv.setText(temp);
                }
            }

            // 复杂业务查询
            public void queryHard(View btn) throws DbException {
                Calendar calendar = Calendar.getInstance();
                calendar.add(Calendar.DATE, -1);
                calendar.add(Calendar.HOUR, 3);
                // 复杂业务示例：条件，排序，分页
                List<Parent> list = db.selector(Parent.class)
                        .where("id", "<", 54)
                        .and("time", ">", calendar.getTime())
                        .orderBy("id")
                        .limit(10) // 只查询10条记录
                        .offset(10) // 偏移10个,从第11个记录开始返回,limit配合offset达到sqlite的limit m,n的查询
                        .findAll();
                temp = "List<Parent> Size:" + list.size() + "\n";
                tv.setText(temp);
                if (list.size() > 0) {
                    temp += "Last Parent Object:" + list.get(list.size() - 1) + "\n";
                    tv.setText(temp);
                }
            }

            // 分组查询
            public void queryGroup(View btn) throws DbException {
                // 根据name分组查询数据
                List<DbModel> dbModels = db.selector(Parent.class)
                        .groupBy("name")
                        .select("name", "count(name) as count").findAll();
                temp = "Group By Result:" + dbModels.get(0).getDataMap() + "\n";
                tv.setText(temp);
            }

            // 批量查询
            public void queryMore(View btn) {
                ORMManager.task().run(new Runnable() { // UI异步执行
                    @Override
                    public void run() {
                        try {
                            long start = System.currentTimeMillis();
                            db.selector(Parent.class).orderBy("id", true).limit(1000).findAll();
                            temp = "查找1000条数据:" + (System.currentTimeMillis() - start) + "ms\n";
                        } catch (DbException ex) {
                            ex.printStackTrace();
                            temp = "批量查询异常";
                        }
                        final String finalResult = temp;
                        ORMManager.task().post(new Runnable() { // UI同步执行
                            @Override
                            public void run() {
                                tv.setText(finalResult);
                            }
                        });
                    }
                });
            }

            // 批量删除
            public void deleteMore(View btn) {
                ORMManager.task().run(new Runnable() { // UI异步执行
                    @Override
                    public void run() {
                        try {
                            List<Parent> parentList = db.selector(Parent.class).orderBy("id", true).limit(1000).findAll();
                            long start = System.currentTimeMillis();
                            db.delete(parentList);
                            temp = "删除1000条数据:" + (System.currentTimeMillis() - start) + "ms\n";
                        } catch (DbException ex) {
                            ex.printStackTrace();
                            temp = "批量删除异常";
                        }
                        final String finalResult = temp;
                        ORMManager.task().post(new Runnable() { // UI同步执行
                            @Override
                            public void run() {
                                tv.setText(finalResult);
                            }
                        });
                    }
                });
            }
        }
