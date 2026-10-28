# NestJS: Service Cannot Find Repository Dependency


# NestJS: Service Cannot Find Repository Dependency

## Cause

The following code:

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

At runtime it reports: `Nest can't resolve dependencies of the SiteRbacService`, and the log shows that `@InjectRepository(RbacEntity) public readonly repository: Repository<RbacEntity>` was not injected.

## Solution

Add the following in `site-rbac.module.ts`:

```typescript
@Module(
  {
    imports: [
      TypeOrmModule.forFeature([RbacEntity]),
    ]
  }
)
```

