# nestjsx/crud xxxRepository not found问题解决


# nestjsx/crud xxxRepository not found问题解决

`nestjsx/crud`可以快速帮我生成   使用`@Entity `定义好`typeorm`对象的对应`service`,`controller`的 `增删改查`操作`API`  . 

但是,使用 `nestjsx/crud`时,总是会有例如: `Nest can't resolve dependencies of the XXXService (?). Please make sure that the argument UtilsService at index [0] is available in the UtilsModule context.`的错误,

其原因一般为,在`XXXModule`中没有`import`对应 `TypeOrm.forFeature([XXXEntity])`,在`exports`及`providers`中没有导出`XXXService`, 

**总要**, 在`app.module.ts`中,同样也需要在`@Muldue`的`imports`中增加`XXXModule`.

注意,如果`XXXModule`是定义在`nestjs`的`library`中,需要在`XXXModule.eports`中增加`TypeOrmModule`,不然在`app`编译运行后,会报错提示`XXXService`中没有导入`XXXEntityRepository`对象.

tags: tag: nestjs, typeorm, entity, nestjsx/crud, can't resolve dependencies

