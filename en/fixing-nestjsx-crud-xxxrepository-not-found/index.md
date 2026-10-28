# Fixing 'nestjsx/crud xxxRepository not found'


# Fixing 'nestjsx/crud xxxRepository not found'

`nestjsx/crud` can quickly generate the `CRUD` operation `APIs` of the corresponding `service` and `controller` for the `typeorm` objects defined with `@Entity`.

However, when using `nestjsx/crud`, there are always errors such as: `Nest can't resolve dependencies of the XXXService (?). Please make sure that the argument UtilsService at index [0] is available in the UtilsModule context.`

The usual cause is that `XXXModule` has not `import`ed the corresponding `TypeOrm.forFeature([XXXEntity])`, and has not exported `XXXService` in `exports` and `providers`.

**Above all**, in `app.module.ts` you likewise need to add `XXXModule` to the `imports` of `@Muldue`.

Note that if `XXXModule` is defined in a `nestjs` `library`, you need to add `TypeOrmModule` to `XXXModule.eports`, otherwise after the `app` is compiled and run, it will report an error saying that the `XXXEntityRepository` object is not imported in `XXXService`.

tags: tag: nestjs, typeorm, entity, nestjsx/crud, can't resolve dependencies

