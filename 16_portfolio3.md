## はじめに
前回はポートフォリオを大改造してデータをデータベースで管理するようにしました。

https://qiita.com/piny940/items/f1a15f7a797c4d0f7461

今回は、前回やり残した「プロジェクトの並び替え」部分を実装した話を記そうと思います。

## 抱えていた問題

前回ポートフォリオのデータをデータベースで管理する方向性にシフトしたのですが、DB設計の段階で考え漏れていたことがありました。それは「プロジェクトの並び替え」です。  
プロジェクトは現状、写真のようにスターがついたものが上に来るようになっているのですが、Starがついたプロジェクト同士 / Startがついていないプロジェクト同士の順序については操作ができない設計になっていました。  

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3330232/6c0689e6-10b5-b54f-df1b-845fa0f4827e.png)

そこで今回は、順序を保存するカラムを追加し、UI上でプロジェクトの並び替えができるようにしていきたいと思います。

## DB設計

順序をデータベース上でどう保存にするかについては、次の3つの方法を案として考えました。

1. レコードに自身の順番(position)をもたせる
2. レコードに自身の優先度(priority)をもたせる
3. レコードに自身の順番を大きな整数でもたせる

各方法についてのメリット・デメリットは以下のようになります。

|方式|メリット|デメリット|
|--|--|--|
|1|実装がシンプル|並び替えのたびに全てのデータを更新する必要がある
|2|並び替え時に大量のデータを更新しなくていい|priorityをすべて手打ちで入力しないといけない|
|3|並び替え時に大量のデータを更新しなくてもいい・並び替えをUIで実装可能|数十回ほど並び替えを繰り返すだけで破綻する

はじめは3の方法がいいと思っていたのですが、positionの値を100,000おきに設定していたとしても、並び替えごとにpositionの値の差は半分になっていきます。

例)
|id|position|
|--|--|
|1|100,000|
|2|200,000|
|3|300,000|

↓
|id|position|
|--|--|
|1|100,000|
|3|150,000|
|2|200,000|

↓
|id|position|
|--|--|
|1|100,000|
|2|125,000|
|3|1500,000|

これを繰り返していくと、ものの数十回ほどでレコード1とレコード2のpositionが同じ値になってしまいます。 
そうなったときに、positionの値はどう割り振り直すのか？などを考えると実装が複雑になってしまいます。

今回の場合は、

- 並び替えは私の気分次第で割と頻繁に行う
- レコードの数は高々数十個程度
- 並び替えはUIを使って手軽に直感的に行えるようにしたい

ということから、1のシンプルな実装にすることにしました。

## フォームのUI

DBの設計が決まったので、次は並び替えのUIをどう作っていくかを検討します。

検討したライブラリは次の4つです。

- [react-dnd](https://github.com/react-dnd/react-dnd/)
- [react-beautiful-dnd](https://github.com/atlassian/react-beautiful-dnd)
- [react-draggable](https://github.com/react-grid-layout/react-draggable)
- [dnd-kit](https://github.com/clauderic/dnd-kit)

それぞれのnpm trendsは写真のとおりでした。  
1年前は`react-draggable`が最も使われていましたが、徐々に減少してきており、最近では`@dnd-kit/core`が伸びてきていることが読み取れます。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3330232/ece9b0d9-28f0-e8d2-6814-3645ab257b96.png)

それぞれのスター数・メンテナンス頻度・書きやすさ(主観)は次のとおりでした。


|ライブラリ|スター数|メンテ頻度|書きやすさ|
|--|--|--|--
|react-dnd|20.2k|1年前|-|
|react-beautiful-dnd|31.8k|7ヶ月前|-|
|react-draggable|8.7k|4ヶ月前|○|
|dnd-kit|10.5k|2ヶ月前|○

`react-dnd`はここ1年メンテナンスがされておらず、`react-beautiful-dnd`はメンテナンスが当面終了と記載されていたので、候補から除外しました。  
`react-draggable`と`dnd-kit`はどちらも良さそうでしたが、
- 日本語の参考サイトの豊富さ
- スター数
- メンテナンス頻度

の観点から`dnd-kit`を採用することにしました。

## 実装
### バックエンド
まず、Graphqlのスキーマは次のようにしました。
idのリストをinputで受け取り、その順序に従ってprojectのpositionを変更するという仕様で、内部のposition等のプロパティは外部で意識しなくてもいいようにしました。
```graphql
input UpdateProjectOrderInput {
  ids: [String!]!
}
extend type Mutation {
  updateProjectOrder(input: UpdateProjectOrderInput!): [Project!]!
}
```

実際の処理部分は次のようになっています(読みやすさのためにエラー処理のコードは省略しています)。  
入力の`ids`に含まれていないプロジェクトの扱いをどうするかは迷ったのですが、指定されていないプロジェクトは順序を保ったまま後ろにずらすという仕様にしました。
`position`のデフォルト値は0にして、デフォルトで1番上に来るようにしたいため、指定するpositionの値は1以上になるようにしました。
```go
allProjects, err := u.repo.List()
newProjects := make([]*domain.Project, len(input.Ids))
for _, project := range allProjects {
  pos := slices.Index(input.Ids, project.ID)
  projectInput := project.ToInput()
  if pos < 0 {
    newPos := project.Position + len(input.Ids)
    projectInput.Position = &newPos
  } else {
    newPos := pos + 1
    projectInput.Position = &newPos
  }
  newProject, err := u.repo.Update(projectInput)
  if pos >= 0 {
    newProjects[pos] = newProject
  }
}
return newProjects, nil
```

### フロントエンド
フロントエンドのコードは概ね[ドキュメント](https://docs.dndkit.com/presets/sortable)の通りです。  
`restrictToVerticalAxis`modifierを使うことで縦方向にしかドラッグできないようにしたり、順序を変えたあとはSaveするまで他の操作が出来ないようにしたりといった細かな工夫を施しています。

<details>
<summary>コード</summary>

```typescript
export const Projects = (): JSX.Element => {
  const context = useMemo(() => ({ additionalTypenames: ['Project'] }), [])
  const [{ data, error }] = useGetProjectsQuery({ context })
  const [, deleteProject] = useDeleteProjectMutation()
  const [projects, setProjects] = useState<ProjectType[]>()
  const [, updateProjectOrder] = useUpdateProjectOrderMutation()
  const [orderChanged, setOrderChanged] = useState(false)

  const onDragEnd = useCallback((event: DragEndEvent) => {
    const { active, over } = event
    if (over && active.id !== over.id) {
      setProjects((projects) => {
        if (!projects) return
        const ids = projects.map((project) => project.id)
        const oldIndex = ids.indexOf(active.id as string)
        const newIndex = ids.indexOf(over.id as string)
        return arrayMove(projects, oldIndex, newIndex)
      })
      setOrderChanged(true)
    }
  }, [])
  const saveOrder = useCallback(async () => {
    if (!projects) return
    const ids = projects.map((project) => project.id)
    const { error } = await updateProjectOrder({ input: { ids } })
    if (error) {
      console.error(error)
      return
    }
    setOrderChanged(false)
  }, [projects, updateProjectOrder])
  const resetOrder = useCallback(() => {
    if (!data) return
    setProjects(data.projects)
    setOrderChanged(false)
  }, [data])

  useEffect(() => {
    if (!data) return
    setProjects(data.projects)
  }, [data])

  if (error) return <Error statusCode={400} />
  if (!projects) return <>loading...</>
  return (
    <Box>
      <Typography variant="h4" component="h2">
        Projects
      </Typography>
      <Box mt={2}>
        <Button
          disabled={orderChanged}
          component={Link}
          href="/projects/new"
          fullWidth
          variant="contained"
        >
          新規作成
        </Button>
      </Box>
      <Box mt={2}>
        <Button disabled={!orderChanged} variant="outlined" onClick={saveOrder}>
          Save Order
        </Button>
        <Button
          sx={{ marginLeft: 1 }}
          variant="outlined"
          disabled={!orderChanged}
          onClick={resetOrder}
        >
          Reset Order
        </Button>
      </Box>
      <DndContext
        collisionDetection={closestCenter}
        modifiers={[
          restrictToVerticalAxis,
          restrictToParentElement,
          restrictToParentElement,
        ]}
        onDragEnd={onDragEnd}
      >
        <SortableContext
          strategy={verticalListSortingStrategy}
          items={projects}
        >
          <Table>
            <TableHead>
              <TableRow>
                <TableCell></TableCell>
                <TableCell>Id</TableCell>
                <TableCell>Title</TableCell>
                <TableCell>Description</TableCell>
                <TableCell>IsFavorite</TableCell>
                <TableCell>Position</TableCell>
                <TableCell>CreatedAt</TableCell>
                <TableCell>UpdatedAt</TableCell>
                <TableCell>Links</TableCell>
              </TableRow>
            </TableHead>
            <TableBody>
              {projects.map((project) => (
                <ProjectItem
                  actionsDisabled={orderChanged}
                  key={project.id}
                  project={project}
                  deleteProject={() => {
                    void deleteProject({ id: project.id })
                  }}
                />
              ))}
            </TableBody>
          </Table>
        </SortableContext>
      </DndContext>
    </Box>
  )
}

type ProjectItemProps = {
  project: ProjectType
  deleteProject: () => void
  actionsDisabled?: boolean
}
const ProjectItem = ({
  project,
  deleteProject,
  actionsDisabled = false,
}: ProjectItemProps) => {
  const { listeners, setNodeRef, transform, transition, attributes } =
    useSortable({
      id: project.id,
    })
  const style = {
    transform: CSS.Transform.toString(transform),
    transition,
  }
  return (
    <TableRow ref={setNodeRef} style={style} {...attributes}>
      <TableCell>
        <IconButton {...listeners}>
          <DragHandleIcon />
        </IconButton>
      </TableCell>
      <TableCell>{project.id}</TableCell>
      <TableCell>{project.title}</TableCell>
      <TableCell>{project.description}</TableCell>
      <TableCell>{project.isFavorite ? 'true' : 'false'}</TableCell>
      <TableCell>{project.position}</TableCell>
      <TableCell>{dateLabel(project.createdAt)}</TableCell>
      <TableCell>{dateLabel(project.updatedAt)}</TableCell>
      <TableCell
        sx={{
          '> *': {
            marginLeft: 1,
            marginTop: 1,
          },
        }}
      >
        <Button
          disabled={actionsDisabled}
          variant="contained"
          component={Link}
          size="small"
          href={`/projects/${project.id}/edit`}
        >
          編集
        </Button>
        <Button
          variant="contained"
          disabled={actionsDisabled}
          size="small"
          onClick={deleteProject}
        >
          削除
        </Button>
      </TableCell>
    </TableRow>
  )
}
```
</details>

実際の挙動はこんな感じです。滑らかに動いてくれていて満足しています。
![5-002301531304722913613459231167146.gif](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3330232/af51f4b2-71b3-b74b-b919-0f7688b30911.gif)

## 最後に
今回はプロジェクトの並び替えができるようDB・UIを整えました。順序を保存するようなテーブルの設計をするのは初めてだったので、良い学びになりました。
UIについても、最終的にかなり直感的に操作できるUIができて満足しています。

次回はお家kubernetes作りに戻ってログ周りを整理していきたいと思います。

## 参考資料

https://zenn.dev/itte/articles/e97002637cd3a6

https://zenn.dev/tara_is_ok/articles/40e1d7b6357ee5

https://tech-blog.talentio.co.jp/entry/2023/03/28/143944

https://docs.dndkit.com

https://zenn.dev/wintyo/articles/d39841c63cc9c9

https://zenn.dev/koharu2739/articles/31c240c5ee5278
