# UE4渲染流程分析   
    分析一下在游戏线程中更改了摄像机参数后是如何生效的，以后处理(PostProcess)为例。
## 游戏线程 
### 如何更改摄像机后处理参数
    对于每一个APlayerController实例来说，都含有一个APlayerCameraManager对象，在APlayerCameraManager中包含有一个TArray<class UCameraModifier*> ModifierList成员变量，最终通过UCameraModifier类对象来修改PlayerCamera的参数。ModifierList变量可以通过继承自APlayerManager的蓝图类中添加。
    ```
    class APlayerController
    {
        ...

    public:
        UPROPERTY(BlueprintReadOnly, Category = PlayerController)
        APlayerCameraManager* PlayerManager;
    };

    class APlayerCameraManager
    {
        ...

    protected:
       UPROPERTY() 
       TArray<class UCameraModifier*> ModifierList;

        ...
    };
    ```
    上面说通过UCameraModifier类对象来修改PlayerCamera的参数并不准确，我们一般通过继承自UCameraModifier的子类来实现想要的效果。以后处理效果中的运动模糊为例：通过重载ModifyCamera函数来实现对PlayerCamera的修改。
    ```
    class UCameraModifierPostProcess : public UCameraModifier
    {
        ...
    public:
        virtual bool ModifyCamera(float DeltaTime, struct FMinimalViewInfo& InOutPov) override
        {
            FPostProcessSettings PostProcessSettings;
            PostProcessSettings.bOverride_FullScreenBlur = true;
            PostProcessSettings.bFullScreenBlur = false;
            ...
            
            InOutPov.PostProcessSettings = PostProcessSettings;

            ...
        }
        ...
    };
    ```
    修改PlayerCamera的整个流程如下所示：最终将ViewTarget的相关信息缓存在APlayerCameraManager的CameraCachePrivate中。
    ```
    APlayerCameraManager::DoUpdateCamera(float DeltaTime)
    {
        FMinimalViewInfo NewPOV = ViewTarget.POV;

        APlayerCameraManager::UpdateViewTarget(FTViewTarget& ViewTarget, float DeltaTime)
        {
            APlayerCameraManager::ApplyCameraModifiers(float DeltaTime, FMinimalViewInfo& OutVT.POV)
            {
                for (int32 ModifierIdx = 0; ModifierIdx < ModifierList.Num(); ++ModifierList)
                {
                    ModifierList[ModifierIdx]->ModifyCamera(DeltaTime, FMinimalViewInfo& OutVT.POV);
                }
            }
        }

        NewPOV = ViewTarget.POV;
        APlayerCameraManager::FillCameraCache(const FMinimalViewInfo& NewPOV)
        {
            CameraCachePrivate.POV = NewPOV;
        }
    }
    ```
### 将更改后信息发送给渲染线程 [参考资料](https://blog.csdn.net/jiangdengc/article/details/60141724) 
    在游戏线程中：
    ```
    void UGameEngine::Tick(float DeltaTime, bool bIdleMode)
    {
        void UGameEngine::RedrawViewports()
        {
           void UGameViewportClient::Draw()
           {
               FSceneViewFamilyContext ViewFamily;
               FSceneView* View = LocalPlayer->CalcSceneView(ViewFamily, ...)
               {
                   // 获取在APlayerCameraManager中更改过的CameraCachePrivate信息
                   ViewFamily.Views[0].ViewInfo = PlayerContoller->PlayerManager->CameraCachePrivate.POV;
               }
               GetRendererModule().BeginRenderingViewFamily(..., &ViewFamily)
               {
                   FSceneRenderer* SceneRenderer = FSceneRenderer::CreateSceneRenderer(ViewFamily);
                   ENQUEUE_UNIQUE_RENDER_COMMAND_ONEPARAMETER(
                        FDrawSceneCommand,
                        FSceneRenderer* SceneRenderer, SceneRenderer,
                    {
                        RenderFamily_RenderThread(RHICmdList, SceneRenderer);
                        FlushPendingDeleteRHIResources_RenderThread();
                    });
               }
           }
        }
    }
    ```
## 渲染线程 
    渲染线程的入口函数还是RenderFamily_RenderThread函数，第一步进行 InitViews()，首先调用 Primitive Visibility Determination 进行剪裁，然后是透明物的排序，然后是灯光的可见性，然后就是不透明物体的排序。
    接下来通过很多的 pass 来实现整个渲染。首先会有一个 base pass，建立一个 base 缓冲，然后通过 base pass，填充 GBuffer 的缓冲，然后是渲染所有的灯光，后面就是渲染天光，渲染大气效果，渲染透明对象，渲染屏幕区特效，所有这些渲染完之后， SceneColor() 就完成了，最后进行后处理，最后是调用 RenderFinish()。
    ```
    void FDeferredShadingSceneRenderer::Render()
    {
        bool FDeferredShadingSceneRenderer::InitViews()
        {
            //– Visibility determination.
            void FSceneRenderer::ComputeViewVisibility()
            {
                FrustumCull();
                OcclusionCull();
            }
            //– 透明对象排序：back to frontFTranslucentPrimSet::SortPrimitives();

            //determine visibility of each lightDoFrustumCullForLights();

            //– Base Pass对象排序：front to back

            void FDeferredShadingSceneRenderer::SortBasePassStaticData();
        }
    
        //– EarlyZPass
        FDeferredShadingSceneRenderer::RenderPrePass();
        RenderOcclusion();
        //– Build Gbuffers
        SetAndClearViewGBuffer(); 
        FDeferredShadingSceneRender::RenderBasePass();
        FSceneRenderTargets::FinishRenderingGBuffer();

        //– Lighting stage
        RenderDynamicSkyLighting();
        RenderAtmosphere();
        RenderFog();
        RenderTranslucency();
        RenderDistortion();
        //– post processing
        SceneContext.ResolveSceneColor();
        FPostProcessing::Process();
        FDeferredShadingSceneRenderer::RenderFinish();
   } 

    void FDeferredShadingSceneRenderer::RenderLights()
    {
        foreach(FLightSceneInfoCompact light IN Scene->Lights)
        {
            void FDeferredShadingSceneRenderer::RenderLight(Light)
            {
                RHICmdList.SetBlendState(Additive Blending);
                // DeferredLightVertexShaders.usf VertexShader = TDeferredLightVS; 
                // DeferredLightPixelShaders.usf PixelShader = TDeferredLightPS;
                switch(Light Type)
                {
                case LightType_Directional:
                    DrawFullScreenRectangle();
                case LightType_Point:
                    StencilingGeometry::DrawSphere();
                case LightType_Spot:
                    StencilingGeometry::DrawCone();
                }
            }
        }
    } 
    ```
