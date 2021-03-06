---
title: "Master Memory事始め(個人開発編)"
emoji: "😎"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [Unity, MasterMemory, csharp]
published: true
---

[「Applibot Advent Calendar 2020」](https://qiita.com/advent-calendar/2020/applibot) 18日目の記事になります。
前日は [@Nakamuro-unl](https://qiita.com/Nakamuro-unl) さんの[1年限定運営アプリゲーム「SEVEN's CODE」の開発思想について](https://qiita.com/Nakamuro-unl/items/adb6706f0ea732fa7a46)という記事でした!

:::details 目次(クリックすると展開されます)
<!-- TOC -->

- [はじめに](#はじめに)
- [概要](#概要)
- [実行環境](#実行環境)
- [準備](#準備)
  - [MasterMemoryの導入](#mastermemoryの導入)
  - [MessagePackの導入](#messagepackの導入)
- [テーブル定義](#テーブル定義)
  - [例](#例)
    - [アノテーション説明](#アノテーション説明)
- [ジェネレータの起動](#ジェネレータの起動)
  - [MasterMemoryのジェネレータ](#mastermemoryのジェネレータ)
    - [例](#例-1)
  - [MessagePackのジェネレータ](#messagepackのジェネレータ)
    - [例](#例-2)
- [閑話休題](#閑話休題)
- [マスタバイナリ作成](#マスタバイナリ作成)
  - [Unity内でマスタデータを作成する](#unity内でマスタデータを作成する)
    - [例](#例-3)
- [Unity側でマスタデータを読み取る処理](#unity側でマスタデータを読み取る処理)
- [まとめ](#まとめ)
- [付録](#付録)
- [参考記事](#参考記事)

<!-- /TOC -->
:::

## はじめに
本記事では[Master Memory](https://github.com/Cysharp/MasterMemory)を個人開発で使用したときのUnity側マスタデータキャッシュについて使い方等を紹介していきます。

## 概要
MasterMemoryはマスターデータの管理を主眼に置いた、読み取り専用のインメモリデータベースです。
採用のメリットデメリットについては[参考記事](#参考記事)をご覧ください。

## 実行環境
- OS: macOS 10.15.6 
- Unity: 2020.1.0.f1
## 準備
MasterMemoryとMessagePackを導入していきます

### MasterMemoryの導入
[Cysharp/MasterMemory](https://github.com/Cysharp/MasterMemory/releases)
上記のページにアクセスして、下記の2つのファイルをダウンロードします
**MasterMemory.Unity.unitypackage**
Unity内部で必要になります。
**MasterMemory.Generator.zip**
dotnet-toolsを用いる場合は不要になります。
dotnetでの導入手順は付録のGitHubにて説明します。

```zip解凍後のディレクトリ
MasterMemory.Generator.zip
├── linux-x64
│   └── MasterMemory.Generator
├── osx-x64
│   └── MasterMemory.Generator
└── win-x64
    └── MasterMemory.Generator.exe
```

### MessagePackの導入
[neuecc/MessagePack-CSharp](https://github.com/neuecc/MessagePack-CSharp/releases)
上記のページにアクセスして、下記の2つのファイルをダウンロードします。
**MessagePack.Unity.2.1.115.unitypackage**
Unity内部で必要になります。
**mpc.zip**
dotnet-toolsを用いる場合は不要にになります。
dotnetでの導入手順は付録のGitHubにて説明予定です。

```zip解凍後のディレクトリ
mpc.zip
├── linux
│   └── mpc
├── osx
│   └── mpc
└── win
    └── mpc.exe
```

## テーブル定義
Master Memoryで使うテーブルの定義ファイルを作成します。
以下の例に示すような型のインスタンスがテーブルで管理されているイメージです。
MasterMemoryでは内部でMessagePackを使っており、バイナリの方式をMap型かArray型で選ぶことができます。(Array型の方がバイナリのサイズを小さくできるため、本記事ではそちらを採用します。)
### 例
テーブル定義クラスの例を以下に示します。

:::details テーブル定義クラス
```csharp: MPokemon.cs (MessagePackのMap型使用)
using System.Collections;
using System.Collections.Generic;
using MasterMemory;
using MessagePack;

/// <summary>
/// MPokemonテーブル
/// </summary>
[MemoryTable("m_pokemon"), MessagePackObject(true)]
public partial class MPokemon
{
    [PrimaryKey] 
    public long Id { get; private set; }

    public string DisplayName { get; private set; }

    public int Hp { get; private set; }

    public int Attack { get; private set; }

    public int Defense { get; private set; }

    public int SpecialAttack { get; private set; }

    public int SpecialDefence { get; private set; }
    
    public int Speed { get; private set; }

    public override string ToString()
    {
        return "Id, DisplayName, Hp, Attack, Defence, SpecialAttack, SpecialDefense, Speed" + "\n" +
               string.Format("{0}, {1}, {2}, {3}, {4}, {5}, {6}, {7}", Id, DisplayName, Hp, Attack, Defense,
                   SpecialAttack, SpecialDefence, Speed);
    }
}
```

```csharp: MPokemon.cs (MessagePackのArray型使用)
using System.Collections;
using System.Collections.Generic;
using MasterMemory;
using MessagePack;

/// <summary>
/// MPokemonテーブル
/// </summary>
[MemoryTable("m_pokemon"), MessagePackObject]
public partial class MPokemon
{
    [Key(0)] 
    [PrimaryKey] 
    public long Id { get; private set; }
    [Key(1)] 
    public string DisplayName { get; private set; }
    [Key(2)] 
    public int Hp { get; private set; }
    [Key(3)] 
    public int Attack { get; private set; }
    [Key(4)] 
    public int Defense { get; private set; }
    [Key(5)] 
    public int SpecialAttack { get; private set; }
    [Key(6)] 
    public int SpecialDefence { get; private set; }
    [Key(7)] 
    public int Speed { get; private set; }

    public override string ToString()
    {
        return "Id, DisplayName, Hp, Attack, Defence, SpecialAttack, SpecialDefense, Speed" + "\n" +
               string.Format("{0}, {1}, {2}, {3}, {4}, {5}, {6}, {7}", Id, DisplayName, Hp, Attack, Defense,
                   SpecialAttack, SpecialDefence, Speed);
    }
}
```
:::
このテーブル定義クラスはPlantUMLやjsonから自動生成されるべきものですが、今回は**個人開発**を隠蓑にして割愛させていただきます。

#### アノテーション説明
**[MemoryTable (“m_pokemon”)]** : MasterMemoryのテーブルとして読み取るためのアノテーション。
**[MessagePackObject(true)]** : MessagePackのオブジェクトとして読み取るためのアノテーション。trueでMap型、falseでArray型、(true/false)を付けないとArray型としてGenerateされます。
**[PrimaryKey]** : プライマリキーを指定するアノテーション。
**[SecondaryKey (hoge), NonUnique]** : セカンダリキーを指定するアノテーション。hogeにはセカンダリインデックスが入る。NonUniqueを指定できるます。
**[SecondaryKey (hoge, keyOrder : fuga), NonUnique]** : 複合キーを指定するアノテーション。keyOrderには複合キーの順序を指定できる。

## ジェネレータの起動
本記事ではUnity EditorからのGenerator起動について書いていきます。
CLIよりGeneratorを起動する場合は付録などを参考にしていただきたいです。
### MasterMemoryのジェネレータ
ジェネレータを起動するコード例を以下に示します。
#### 例
:::details MasterMemoryのGenerator起動クラス
```csharp: MasterMemoryGenerator.cs
using System.Diagnostics;
using UnityEditor;
using UnityEngine;

public class MasterMemoryGenerator
{
    [MenuItem("MasterMemory/CodeGenerate")]
    private static void GenerateMasterMemory()
    {
        ExecuteMasterMemoryCodeGenerator();
    }

    private static void ExecuteMasterMemoryCodeGenerator()
    {
        UnityEngine.Debug.Log($"{nameof(ExecuteMasterMemoryCodeGenerator)} : start");

        var exProcess = new Process();

        var rootPath = Application.dataPath + "/..";
        var filePath = rootPath + "/GeneratorTools/MasterMemory.Generator";
        var exeFileName = "";
#if UNITY_EDITOR_WIN
        exeFileName = "/win-x64/MasterMemory.Generator.exe";
#elif UNITY_EDITOR_OSX
        exeFileName = "/osx-x64/MasterMemory.Generator";
#elif UNITY_EDITOR_LINUX
        exeFileName = "/linux-x64/MasterMemory.Generator";
#else
        return;
#endif
        var psi = new ProcessStartInfo()
        {
            CreateNoWindow = true,
            WindowStyle = ProcessWindowStyle.Hidden,
            RedirectStandardOutput = true,
            RedirectStandardError = true,
            UseShellExecute = false,
            FileName = filePath + exeFileName,
            // TODO: 使用する場合はPath, Argumentsを変更してください
            Arguments =
                $@"-i ""{Application.dataPath}/Scripts/TableDefines"" -o ""{Application.dataPath}/Scripts/Generated/MasterMemory"" -c -n Generated",
        };

        var p = Process.Start(psi);

        p.EnableRaisingEvents = true;
        p.Exited += (object sender, System.EventArgs e) =>
        {
            var data = p.StandardOutput.ReadToEnd();
            UnityEngine.Debug.Log($"{data}");
            UnityEngine.Debug.Log($"{nameof(ExecuteMasterMemoryCodeGenerator)} : end");
            p.Dispose();
            p = null;
        };
    }
}
```
:::
ProcessStartInfo内のArgumentsにてオプションを設定することができます。
よく使ういくつかのオプションを紹介していきます。
**-i** (必須)インプットディレクトリの指定。先ほど作成したテーブル定義クラスを格納しているディレクトリを指定してください。
**-o** (必須)アウトプットディレクトリの指定。Generatorによる自動生成コードの出力先ディレクトリを指定してください。
**-n** (必須)出力したクラスの名前空間指定です。
**-c** テーブル定義クラスにコンストラクタを追加するかの指定です。今回はテーブル定義クラスのプロパティのsetをprivateにしているため、追加しておきます。
**-p** 生成後クラスに独自の接頭辞をつけるかの指定です。つける場合は -p "Hoge"などで指定できます。

### MessagePackのジェネレータ
ジェネレータを起動するコード例を以下に示します。
#### 例
:::details MessagePackGenerator.cs
```csharp: MessagePackGenerator.cs
using System.Diagnostics;
using UnityEditor;
using UnityEngine;

public class MessagePackGenerator
{
    [MenuItem("MessagePack/CodeGenerate")]
    private static void GenerateMessagePack()
    {
        ExecuteMessagePackCodeGenerator();
    }

    private static void ExecuteMessagePackCodeGenerator()
    {
        UnityEngine.Debug.Log($"{nameof(ExecuteMessagePackCodeGenerator)} : start");

        var exProcess = new Process();

        var rootPath = Application.dataPath + "/..";
        var filePath = rootPath + "/GeneratorTools/mpc";
        var exeFileName = "";
#if UNITY_EDITOR_WIN
        exeFileName = "/win/mpc.exe";
#elif UNITY_EDITOR_OSX
        exeFileName = "/osx/mpc";
#elif UNITY_EDITOR_LINUX
        exeFileName = "/linux/mpc";
#else
        return;
#endif

        var psi = new ProcessStartInfo()
        {
            CreateNoWindow = true,
            WindowStyle = ProcessWindowStyle.Hidden,
            RedirectStandardOutput = true,
            RedirectStandardError = true,
            UseShellExecute = false,
            FileName = filePath + exeFileName,
            // TODO: 使用する場合はPathを変更してください
            Arguments =
                $@"-i ""{Application.dataPath}/Scripts/TableDefines"" -o ""{Application.dataPath}/Scripts/Generated/MessagePack""",
        };

        var p = Process.Start(psi);

        p.EnableRaisingEvents = true;
        p.Exited += (object sender, System.EventArgs e) =>
        {
            var data = p.StandardOutput.ReadToEnd();
            UnityEngine.Debug.Log($"{data}");
            UnityEngine.Debug.Log($"{nameof(ExecuteMessagePackCodeGenerator)} : end");
            p.Dispose();
            p = null;
        };
    }
}
```
:::
ProcessStartInfo内のArgumentsにてオプションを設定することができます。
よく使ういくつかのオプションを紹介していきます。
**-i** (必須)インプットディレクトリまたはcsprojの指定。先ほど作成したテーブル定義クラスを格納しているディレクトリまたはcsprojを指定してください。
**-o** (必須)アウトプットディレクトリの指定。Generatorによる自動生成コードの出力先ディレクトリを指定してください。
**-n** (必須)出力したクラスの名前空間指定です。

## 閑話休題
UnityでMasterMemoryを扱うための準備は一通り終わりです。
今後の流れとしてはマスタデータのバイナリを作成し、Unity側で受け口となるコードを書いたらMasterMemoryを使えるようになります。

## マスタバイナリ作成
本来、CSVから生成するのが理想ですが、個人開発のためUnity内でマスタデータのバイナリを作成します。

### Unity内でマスタデータを作成する
Editorより、マスタデータを直に入力してマスタデータを作成します。
#### 例
:::details マスタデータバイナリ作成
```csharp: MasterDataGenerator.cs
using System.IO;
using MessagePack.Resolvers;
// MasterMemory.Generatorにて生成時にNameSpaceをGeneratedで指定
using Generated;
using MessagePack;
using UnityEditor;
using UnityEngine;

public static class MasterDataGenerator
{
    [MenuItem("MasterMemory/MasterDataGenerator")]
    static void BuildMasterData()
    {
        // MessagePackのResolverを設定
        try
        {
            StaticCompositeResolver.Instance.Register
            (
                new IFormatterResolver[]
                {
                    MasterMemoryResolver.Instance,
                    GeneratedResolver.Instance,
                    StandardResolver.Instance,
                });
            var options = MessagePackSerializerOptions.Standard.WithResolver(StaticCompositeResolver.Instance);
            MessagePackSerializer.DefaultOptions = options;
        }
        catch
        {
        }

        // 本題のMasterデータ作成はこちら
        var builder = new DatabaseBuilder();
        builder.Append(new MPokemon[]
        {
            new MPokemon(Id: 1, DisplayName: "フシギダネ", Hp: 45, Attack: 49, Defense: 49, SpecialAttack: 65,
                SpecialDefence: 65, Speed: 45),
            new MPokemon(Id: 6, DisplayName: "リザードン", Hp: 78, Attack: 84, Defense: 78, SpecialAttack: 109,
                SpecialDefence: 85, Speed: 100),
        });

        byte[] data = builder.Build();
        Debug.Log("ConvertToJson : " + MessagePackSerializer.ConvertToJson(data));
        var resourcesDir = $"{Application.dataPath}/Resources";
        Directory.CreateDirectory(resourcesDir);
        var filename = "/master-data.bytes";

        using (var fs = new FileStream(resourcesDir + filename, FileMode.Create))
        {
            fs.Write(data, 0, data.Length);
        }

        Debug.Log($"Write byte[] to: {resourcesDir + filename}");

        AssetDatabase.Refresh();
    }
}
```
:::
本コードではマスタデータを配列として入力し、バイナリとして書き出しています。
バイナリのフォーマットはMessagePackです。
そのため、MessagePackのResolverを登録する必要があります。

## Unity側でマスタデータを読み取る処理
マスタデータで読み取る側でもMessagePackのResolverを登録する必要があります。
また、マスタデータの実バイナリはResourcesに格納されているため、持ってくる必要もあります。
Resolverの登録例を以下に示します。
:::details MessagePackResolverの登録
```csharp: Initializer.cs
using Generated;
using MessagePack.Resolvers;
using MessagePack;
using UnityEngine;
 
public static class Initializer {
    [RuntimeInitializeOnLoadMethod (RuntimeInitializeLoadType.BeforeSceneLoad)]
    public static void Initialize () {
        StaticCompositeResolver.Instance.Register
        (
            MasterMemoryResolver.Instance,
            GeneratedResolver.Instance,
            StandardResolver.Instance
        );
 
        var options = MessagePackSerializerOptions.Standard.WithResolver( StaticCompositeResolver.Instance );
        MessagePackSerializer.DefaultOptions = options;
    }
}
```
:::

また、実際に使う一例を示します。（持ってくることだけを考えたコードなのでそのまま使うべきではないです）

:::details マスタデータをダウンロードするクラス
```csharp: MasterDownloader.cs
using UnityEngine;
using Generated;

// TODO: よしなに変更してください
public class MasterDownloader
{
    private static MemoryDatabase _db;

    public static MemoryDatabase DB => _db;

    public static void DownloadMasterData()
    {
        _db = new MemoryDatabase((Resources.Load("master-data") as TextAsset).bytes);
    }
}
```
:::
実際はs3などにあるマスタデータをダウンロードするクラスです。
今回はResourcesにバイナリを置いているためクラスにするまでもありません。

:::details 実際使うときのクラス
```csharp: MasterTest.cs
using UnityEngine;

public class MasterTest : MonoBehaviour
{
    // Start is called before the first frame update
    void Start()
    {
        MasterDownloader.DownloadMasterData();
        MPokemon temp = MasterDownloader.DB.MPokemonTable.FindById(6);
        Debug.Log(temp.ToString());
    }
}
```
:::
**[PrimaryKey]** アノテーションなどで設定したプロパティで検索をかけることができます。

## まとめ
今回はMasterMemoryの導入について執筆させていただきました。
MasterMemoryによってマスタデータキャッシュ基盤の作成が楽になりました。
テーブル定義の自動生成や、CSVからのバイナリ作成ツールについては今後記事として書こうと思います。

## 付録
[MasterMemoryTest - github](https://github.com/kozuka-hayato-ab/MasterMemoryTest)

## 参考記事
[MasterMemory – Unityと.NET Coreのための読み取り専用インメモリデータベース - Cygames Engineers' Blog](https://tech.cygames.co.jp/archives/3269/)
[【Unity】MasterMemory の基本的な使い方 - コガネブログ](https://baba-s.hatenablog.com/entry/2020/05/07/090000)