# nestjs中,Service无法找到Repository依赖.md


# nestjs中,Service无法找到Repository依赖

## 启因

如下代码:

```typescript

@Injectable()
export class SiteRbacService {
  constructor(
    @InjectRepository(RbacEntity)
    public readonly repository: Repository<RbacEntity>,
  ) {}
  create(createRbacDto: CreateRbacDto) {
    return 'This action adds a new rbac';
  }

  findAll() {
    return `This action returns all rbac`;
  }

  findOne(id: number) {
    return `This action returns a #${id} rbac`;
  }

  update(id: number, updateRbacDto: UpdateRbacDto) {
    return `This action updates a #${id} rbac`;
  }

  remove(id: number) {
    return `This action removes a #${id} rbac`;
  }
}
```

运行时会报错: `Nest can't resolve dependencies of the SiteRbacService`,日志显示就是: `@InjectRepository(RbacEntity) public readonly repository: Repository<RbacEntity>` 这个没有引入

## 解决方法

在 `site-rbac.module.ts`中增加:

```typescript
@Module(
  {
    imports: [
      TypeOrmModule.forFeature([RbacEntity]),
    ]
  }
)
```


