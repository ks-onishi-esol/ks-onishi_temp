## 環境
使用プラグイン: https://plugins.jenkins.io/github-pullrequest/
動作確認ジョブ: http://ep-tech-ci3:8086/job/Prototype_JOB/job/ks-onishi/job/preMergeTests/job/SDK/job/preMergeTests
DSL の設定値は http://ep-tech-ci3:8086/plugin/job-dsl/api-viewer/index.html#path/pipelineJob　のこの辺りを参考
http://ep-tech-ci3:8086/plugin/job-dsl/api-viewer/index.html#method/hudson.triggers.Trigger$$List.githubPullRequests

## DSL 設定例

```jenkins
pipelineTriggers {
    triggers {
        githubPullRequests {
            triggerMode('CRON')
            spec('* * * * *') // Polling schedule
            preStatus(false) // Update the GitHub PR status to PENDING once the build is queued
            cancelQueued(true) // Cancel pending builds in the queue for the same PR
            abortRunning(false) // Abort running builds for the same PR
            skipFirstRun(true) // Skip older PRs on the first run
            events {
                Open() // Triggers a build of a pull request when the pull request is opened or reopened.
                commitChanged() // Triggers when a previously built pull request's hash has changed from the previous state (i.e., a new commit is pushed, or force-pushed).
            }
            branchRestriction {
                targetBranch('main') // Restrict to specific branches if needed
            }

        }
    }
}
```

## オプションの仕様確認
#### Set status before build
#### 仕様
* pull request からジョブが開始された場合、checks に ジョブのステータスをpending で表示できる。
#### 設定値
* false
* このオプションでは無く、pull request のルールでジョブのステータスを取得中というのが出来るそうなので、それで対応するため不要
### Abort running builds
#### 仕様
* 同一pull request がトリガーとなっているビルドが実行中で存在する場合、既存のビルドを中断し最新のビルドを実行するオプション
#### 仕様の動作確認
* PR-A トリガーの ビルド #1 実行後、 PR-A に対して修正をコミット後に ビルド #2 が実行されると #1 は中断されました。
* PR-Aトリガーの ビルド #1 実行後中にPR-BをOpen後のビルド #2 が走っても ビルド #1は中断されないので別PRによる中断はありません。
#### 設定値
* false

### Cancel queued builds
#### 仕様
* 同一 pull request がトリガーとなっているビルドが queueに積まれている場合、既存の ビルドqueue を破棄し、最新ビルドのみを queue に残すオプション
#### 仕様の動作確認
* ※同時実行を無効、既存のジョブを動かしている状態で確認
* PR-A を起票後、queue-1 が作成されました。この状態で PR-A に更新コミットを加えると数は増えませんでしたがqueue-1 が置き換わったような表示になりましした。 queue に入ったジョブ実行後のコミットハッシュを確認すると更新コミットの値が入っているので、queue の中身だけが入れ替わり、初回のqueueは破棄されていました。
* PR-A を起票後、queue-1 が作成された状態で、PR-B を起票すると queue-2 が作成されました。別のpull requestに対してはqueueの破棄は行わないようです。
#### 設定値
* true

### Skip older PRs on first run
#### 仕様
設定したジョブの初回実行時に、既に起票してある PR に対しては動作を行わない。今後、ビルドトリガ―に該当するpull request 起票や commit change が入った際にビルドが開始される。
#### 仕様の動作確認
* 一度ビルドが実行されると、チェックが外れるが問題なし。
#### 設定値
* true

### Trigger Events
複数あるが、現在は以下の２つが有効
* Pull Request Opened 
  * Draft pull request でもビルドがトリガーされる
    * Draft から正式なpull request に変更するため、ready for merge を押してもビルドは開始されない。(Open や Commit changeには当たらない)
  * 一度closeしたpull requestを再Openに対してもビルドが走る。
    * Open(ビルド開始) -> Close -> 再Open(ビルド開始)
* Commit changed
  * pull request のマージ元ブランチへコミットすると、自動でpull request に反映され、ビルドが開始される。

### Experimental: User Restriction
あまり調べられていないが、ユーザーの制限は今の所必要ないのでは。

### Experimental: Branch Restriction
#### 仕様
pull request のポーリング対象となるブランチ指定
マージ先のブランチを書く模様
複数かけそうなのだが、複数書くと動かせていない。

Whitelist Target Branches: