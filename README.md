## About

Pool short-lived objects like projectiles or long-lived objects like environment props or entities.

Checkout the [Features](#features) section below for more information about the specific use cases and available helper scripts.

### Requirements

You need to understand the concepts of inheritance and generics in C# to use this package.

## Support

Since I am developing and maintaining this asset package in my spare time, feel free to support me [via paypal](https://paypal.me/NikosProjects), [buy me a coffee](https://ko-fi.com/nikocreatestech) or check out my [games](https://store.steampowered.com/curator/44541812) or other [published assets](https://assetstore.unity.com/publishers/52812).

## Documentation

See the API doc [here](http://pooling.api.niko-creates.tech/annotated.html)

## Setup

### Unity Package Dependency

To add this toolkit as a package dependency to your Unity project, locate your manifest file in "Package/manifest.json" or add the git-url via the package manager UI.

In the previous versions of this package you had to add the NaughtyAttributes package dependency to the "scopedRegistries". Unfortunately this forced you to use a specific fork or version, so to avoid that restriction you have to add the NaughtyAttributes git url (fork/ version) of your liking yourself. The current dependency is a fork with performance improvements ([https://github.com/nikodemgrz/NaughtyAttributes](https://github.com/nikodemgrz/NaughtyAttributes)) of the original open-source project NaughtyAttributes by dbrizov: [https://github.com/dbrizov/NaughtyAttributes](https://github.com/dbrizov/NaughtyAttributes) 

The original NaughtyAttributes package works as well though and if you already have it installed, you don't have to add the forked branch in the following steps!

Add the following git urls in the Unity PackageManager:
```
"https://github.com/nikodemgrz/Unity3DHelperTools.git#upm"
"https://github.com/nikodemgrz/NikosAssets.Pooling.git"
```

For my NaughtyAttributes performance improvements fork:
```
"https://github.com/nikodemgrz/NaughtyAttributes.git#upm"
```

The original branch:
```
"https://github.com/dbrizov/NaughtyAttributes.git#upm"
```

You can also choose specific releases and tags after the "#" instead of "upm".

## Features

Examples for short-lived objects you want to pool, in this case money that will pool after collecting or after it despawned.

The money object:

```
public class MoneyItem : BaseNotesMono, IPoolItemOnClock
{
    public event Action<IPoolItem> OnDestroyed;

    [SerializeField]
    [BoxGroup(HelperConstants.ATTRIBUTE_FIELD_BOXGROUP_SETTINGS)]
    private MoneyItemData _moneyItemData;

    private MoneyManager _moneyManager;
    private PlayerMoneyModule _playerMoneyModule;

    public TimingHelper PoolCooldownToDeactivate { get; set; } = new TimingHelper(TimingHelper.TimerType.Never);
    
    public float Money => _moneyItemData.money;

    public MoneyManager.MoneyType MoneyType => _moneyItemData.moneyType;

    public bool IsOccupied => isActiveAndEnabled;
    public bool IsUsedByMarkers { get; set; }

    private void Awake()
    {
        PoolCooldownToDeactivate = (TimingHelper) _moneyItemData.moneyDisappearsTimer.Clone();
        PoolCooldownToDeactivate.ResetRunningTime();
        PoolCooldownToDeactivate.Init();
        
        _playerMoneyModule = Global.Instance.PlayerContainer
            .PlayerCompositeExecutor.GetModulesByModuleType<PlayerMoneyModule>().FirstOrDefault();
    }

    private void OnDestroy()
    {
        OnDestroyed?.Invoke(this);
    }
    
    public void CollectMoney()
    {
        _playerMoneyModule.AddMoney(Money);
        Deactivate();
    }
    
    public void Deactivate()
    {
        PoolCooldownToDeactivate.ResetRunningTime();
        gameObject.SetActive(false);
    }

    public void PoolingReset()
    {
        PoolCooldownToDeactivate.ResetRunningTime();
        gameObject.SetActive(true);
    }
}
```

The money pool manager containing different pool containers for different money types:

```
public class MoneyManager : BaseNotesMono
{
    public enum MoneyType
    {
        Small = 0,
        Medium = 1,
        Big = 2
    }

    [SerializeField]
    [BoxGroup(HelperConstants.ATTRIBUTE_FIELD_BOXGROUP_SETTINGS)]
    private BaseAlarmClock _checkWorldMoneyAlarm;

    [SerializeField]
    [BoxGroup(HelperConstants.ATTRIBUTE_FIELD_BOXGROUP_SETTINGS)]
    private List<MoneyItem> _moneyItemPrefabs = new List<MoneyItem>();
    
    private Dictionary<MoneyType, PoolContainerOnClock<MoneyItem>> _poolContainerDict = new Dictionary<MoneyType, PoolContainerOnClock<MoneyItem>>();

    private void Awake()
    {
        foreach (MoneyItem moneyItemPrefab in _moneyItemPrefabs)
        {
            PoolContainerOnClock<MoneyItem> poolContainer = new PoolContainerOnClock<MoneyItem>(_checkWorldMoneyAlarm);
            _poolContainerDict[moneyItemPrefab.MoneyType] = poolContainer;
            
            poolContainer.Init();
        }
    }

    private void OnDestroy()
    {
        foreach (var keyValuePair in _poolContainerDict)
            keyValuePair.Value.Dispose();
    }

    public void DropMoneyAt(Vector3 pos, MoneyType moneyType)
    {
        PoolContainerOnClock<MoneyItem> poolContainer = _poolContainerDict[moneyType];
        if (!poolContainer.TryReusePoolItem(out MoneyItem moneyItem, mi => mi.MoneyType == moneyType))
        {
            moneyItem = MonoBehaviour.Instantiate(_moneyItemPrefabs.Find(mi => mi.MoneyType == moneyType));
            //by adding this item, the despawn timer/ logic will be handled by the poolcontainer. you can also do it externally if you prefer and add / remove this pool item yourself at the desired times.
            poolContainer.AddItem(moneyItem);
        }
        
        moneyItem.transform.position = pos;
    }
}
```

The setup for the long-lived poolable objects is a bit more complicated.

- The ```BasePoolItemMarker``` is the Transform placeholder where you want your pooled environment prop/ object to be placed ```OnEnable``` or set free ```OnDisable``` called by this marker. You must set the key in order for this to work!
- The ```BasePoolItemPreview``` is just a useful tool to setup your markers in the Editor automatically. Using the ```EditorPoolMarkerBaker``` the ```BasePoolItemPreview``` will be replaced by a ```BasePoolItemMarker``` with the key automatically set.
- The ```BasePoolMarkerManager``` is required to handle the pooling or instantiation of your ```IPoolItem``` with the matching key setup in this manager and the ```BasePoolItemMarker```.


Here are some example code snippets for environment props, starting with the marker (spawn spot for the poolable item):

```
public abstract class BaseEnvironmentPoolItemMarker<TPoolMarkerManager, TPoolItemMarker> : BasePoolItemMarker<TPoolMarkerManager, TPoolItemMarker>
where TPoolMarkerManager : BasePoolMarkerManagerWrapper<TPoolMarkerManager, TPoolItemMarker>
where TPoolItemMarker : BasePoolItemMarker<TPoolMarkerManager, TPoolItemMarker>
{
    protected BaseBiomeItem _poolBiomeItem;

    [ShowNativeProperty]
    protected bool IsOccupied { get; set; }
    
    protected SubSceneToOwnerChunkReference _chunkReference;
    [ShowNativeProperty]
    public SubSceneToOwnerChunkReference ChunkReference
    {
        get
        {
            if (_chunkReference == null)
                _chunkReference = transform.root.GetComponent<SubSceneToOwnerChunkReference>();

            return _chunkReference;
        }
    }

    protected override void OnEnable()
    {
        _poolBiomeItem = ((EnvironmentPoolItem) DesignatedPoolItem).BaseBiomeItem;
        IsOccupied = true;
        
        ChunkReference.OnLocalSubScenesLoaded += OnLocalSubScenesLoaded;
        if (_chunkReference.SubChunksLoaded)
            OnLocalSubScenesLoaded();
    }

    protected override void OnDisable()
    {
        _chunkReference.OnLocalSubScenesLoaded -= OnLocalSubScenesLoaded;
        _poolBiomeItem = null;
        IsOccupied = false;
    }
    
    protected void OnLocalSubScenesLoaded()
    {
        if (!IsOccupied)
            return;

        _poolBiomeItem.isPooled = true;
        _poolBiomeItem.keepAlive = true;
        _poolBiomeItem.transform.CopyGlobalTransformValuesFrom(transform);
        
        DesignatedPoolItem.PoolingReset();
        _chunkReference.Chunk.RegisterLateBiomeItem(_poolBiomeItem, false);
    }
}
```

```
public class EntityPoolItemMarker : BaseEnvironmentPoolItemMarker<EntityPoolItemMarkerManager, EntityPoolItemMarker>
{
    protected override void OnEnable()
    {
        EntityPoolItemMarkerManager.Instance.PoolMarkerEnabled(this);
        base.OnEnable();
    }

    protected override void OnDisable()
    {
        EntityPoolItemMarkerManager.Instance.PoolMarkerDisabled(this, DesignatedPoolItem);
        base.OnDisable();
    }
}
```

The marker manager you provide in your scene: 
```
public class EntityPoolItemMarkerManager : BasePoolMarkerManager<SimplePoolItemMono, PoolContainer<SimplePoolItemMono>, EntityPoolItemMarker, EntityPoolItemMarkerManager>
{
}
```

The pool item you place on your poolable object/ prefab:

```
public class EnvironmentPoolItem : SimplePoolItemMono
{
    protected bool _isOccupied = false;
    [ShowNativeProperty]
    public override bool IsOccupied => _isOccupied;

    public override void PoolingReset()
    {
        base.PoolingReset();
        _isOccupied = true;
    }

    public override void Deactivate()
    {
        //probably because the world is unloading - useful in some cases but here just an example/ reminder that it exists as long as the base OnDestroy is called!
        if (CancellationTokenSource.IsCancellationRequested)
            return;

        _isOccupied = false;
        base.Deactivate();
    }
}
```

The preview for the automated Editor setup, in case you don't want to setup the markers manually:

```
public class EntityPoolItemPreview : BasePoolItemPreview
{
}
```
