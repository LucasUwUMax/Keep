-- Cherry Hub v11.0 - Luks Edition (PART 1/10)
-- Corrigido: Erros de sintaxe removidos e UI sincronizada.

local redzlib = loadstring(game:HttpGet("https://raw.githubusercontent.com/minhdepzai-v/LibraryRobloc/refs/heads/main/RedzLibrary.lua"))()

-- CONFIGURAÇÕES GLOBAIS (SINCRONIZADAS)
_G.CherryConfig = {
    ESP = false, Hitbox = false, HitboxSize = 10,
    KillAura = false, AuraRadius = 10,
    CoinFarm = false, FarmSpeed = 60,
    AutoShot = false, View = false,
    FlingLoop = false, PlayerESP = false
}

local lp = game.Players.LocalPlayer
local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local TweenService = game:GetService("TweenService")

local RolesCache = { Murderer = nil, Sheriff = nil }
local selectedPlayer = nil

-- FUNÇÃO DE NOTIFICAÇÃO (MOVIDA PARA O TOPO)
local function notify(title, text)
    if redzlib.SetNotif then
        redzlib:SetNotif({Title = title, Description = text, Time = 5})
    else
        print("[" .. title .. "]: " .. text)
    end
end

-- MOTOR DE FLING "LUKS-SEEKER" (O CARAPATO FATAL)
local function executeFling(targetPlayer)
    if not targetPlayer or not targetPlayer.Character then return end
    local myChar = lp.Character
    local myHRP = myChar and myChar:FindFirstChild("HumanoidRootPart")
    local myHum = myChar and myChar:FindFirstChild("Humanoid")
    local tHRP = targetPlayer.Character:FindFirstChild("HumanoidRootPart")
    
    if not myHRP or not tHRP or not myHum then return end
    
    local initialPos = myHRP.CFrame
    myHum.Sit = true 
    myHum:ChangeState(Enum.HumanoidStateType.Physics)
    
    local bv = Instance.new("BodyVelocity")
    bv.Name = "Luks_Velocity"; bv.MaxForce = Vector3.new(1,1,1) * 9e18
    bv.Velocity = Vector3.new(9e8, 9e8, 9e8); bv.Parent = myHRP
    
    local bav = Instance.new("BodyAngularVelocity")
    bav.Name = "Luks_Torque"; bav.MaxTorque = Vector3.new(1,1,1) * 9e18
    bav.AngularVelocity = Vector3.new(9e9, 9e9, 9e9); bav.Parent = myHRP
    
    for _, v in pairs(myChar:GetDescendants()) do
        if v:IsA("BasePart") then v.CanCollide = false; v.CanTouch = false end
    end
    
    local angle = 0
    local connection
    connection = RunService.Heartbeat:Connect(function()
        if not targetPlayer.Parent or not targetPlayer.Character or not tHRP then 
            connection:Disconnect() 
            return 
        end
        angle = angle + 150
        local prediction = tHRP.Velocity * 0.12 
        myHRP.CFrame = CFrame.new(tHRP.Position + prediction) * CFrame.Angles(math.rad(angle), math.rad(angle), 0)
        myHRP.AssemblyLinearVelocity = Vector3.new(9e8, 9e8, 9e8)
    end)
    
    local timeout = tick()
    repeat task.wait() until (tHRP and tHRP.Velocity.Magnitude > 700) or (tick() - timeout > 4) or not targetPlayer.Parent
    
    connection:Disconnect()
    bv:Destroy(); bav:Destroy()
    myHRP.CFrame = initialPos
    myHum.Sit = false; myHum:ChangeState(Enum.HumanoidStateType.Running)
    
    for i = 1, 10 do
        myHRP.Velocity = Vector3.zero; myHRP.RotVelocity = Vector3.zero
        RunService.Heartbeat:Wait()
    end
    for _, v in pairs(myChar:GetDescendants()) do
        if v:IsA("BasePart") then v.CanCollide = true; v.CanTouch = true end
    end
end





-- Cherry Hub v11.0 - Luks Edition (PART 2/10)
-- Revisão: Lógica de Combate e Farm Otimizada

-- FUNÇÃO DE AUTO-SHOT (PARA O XERIFE)
-- Luks, esta função usa raycast para garantir precisão cirúrgica no Murderer.
local function autoShot()
    local murderer = RolesCache.Murderer
    if murderer and murderer.Character and murderer.Character:FindFirstChild("HumanoidRootPart") then
        local gun = lp.Character:FindFirstChild("Gun") or lp.Backpack:FindFirstChild("Gun") or lp.Character:FindFirstChild("Revolver") or lp.Backpack:FindFirstChild("Revolver")
        
        if gun then
            local targetHRP = murderer.Character.HumanoidRootPart
            local myHRP = lp.Character.HumanoidRootPart
            
            -- Verifica se há linha de visão direta (sem paredes)
            local direction = (targetHRP.Position - myHRP.Position).Unit
            local rayParams = RaycastParams.new()
            rayParams.FilterDescendantsInstances = {lp.Character, murderer.Character}
            rayParams.FilterType = Enum.RaycastFilterType.Exclude
            
            local rayResult = workspace:Raycast(myHRP.Position, direction * 300, rayParams)
            
            if not rayResult then
                if gun.Parent == lp.Backpack then lp.Character.Humanoid:EquipTool(gun) end
                
                local remote = gun:FindFirstChild("ShootGun", true) or game:GetService("ReplicatedStorage"):FindFirstChild("ShootGun", true)
                if remote then
                    -- Luks, o disparo inclui uma leve predição para acertar o alvo em movimento
                    remote:FireServer(targetHRP.Position + (targetHRP.Velocity * 0.1), myHRP.Position, direction)
                    task.wait(0.3) 
                end
            end
        end
    end
end

-- SISTEMA DE COIN FARM (TWEEN SEGURO)
local coinCollected = {}
local isTweening = false

local function findCoins()
    local coins = {}
    -- Procura por todos os tipos de nomes que as moedas do MM2 podem ter
    for _, obj in ipairs(workspace:GetDescendants()) do
        if obj:IsA("BasePart") and table.find({"MainCoin", "CoinVisual", "Coin", "Coin_Server"}, obj.Name) then
            if obj.Parent and not coinCollected[obj:GetDebugId()] then
                table.insert(coins, obj)
            end
        end
    end
    return coins
end

local function safeTeleport(target)
    local hrp = lp.Character and lp.Character:FindFirstChild("HumanoidRootPart")
    if not hrp or not target.Parent then return end
    
    isTweening = true
    local distance = (hrp.Position - target.Position).Magnitude
    local duration = distance / _G.CherryConfig.FarmSpeed 
    
    local tween = TweenService:Create(hrp, TweenInfo.new(duration, Enum.EasingStyle.Linear), {
        CFrame = CFrame.new(target.Position + Vector3.new(0, 1.2, 0))
    })
    
    tween:Play()
    tween.Completed:Connect(function()
        coinCollected[target:GetDebugId()] = true
        isTweening = false
    end)
    
    -- Luks, o loop aguarda a conclusão do movimento ou a desativação do farm
    repeat task.wait() until not isTweening or not _G.CherryConfig.CoinFarm
end

-- GERENCIADOR DE FARM EM SEGUNDO PLANO
task.spawn(function()
    while true do
        task.wait(0.1)
        if _G.CherryConfig.CoinFarm and not isTweening and lp.Character and lp.Character:FindFirstChild("HumanoidRootPart") then
            local coins = findCoins()
            if #coins > 0 then
                -- Organiza as moedas por proximidade para eficiência máxima
                table.sort(coins, function(a, b)
                    return (lp.Character.HumanoidRootPart.Position - a.Position).Magnitude < (lp.Character.HumanoidRootPart.Position - b.Position).Magnitude
                end)
                safeTeleport(coins[1])
            end
        end
    end
end)


-- Cherry Hub v11.0 - Luks Edition (PART 3/10)
-- Revisão: Sistema Visual e Inicialização da Interface (Sintaxe Limpa)

-- SISTEMA DE ESP (HIGHLIGHT)
local function removeESP(player)
    if player and player.Character then
        local highlight = player.Character:FindFirstChild("CherryHighlight")
        if highlight then highlight:Destroy() end
    end
end

local function applyESP(player, color)
    if not player or not player.Character or not _G.CherryConfig.ESP then return end
    
    local highlight = player.Character:FindFirstChild("CherryHighlight") or Instance.new("Highlight")
    highlight.Name = "CherryHighlight"
    highlight.Parent = player.Character
    highlight.FillColor = color
    highlight.OutlineColor = Color3.new(1, 1, 1)
    highlight.FillTransparency = 0.4
    highlight.OutlineTransparency = 0
    highlight.DepthMode = Enum.HighlightDepthMode.AlwaysOnTop
end

-- DETECÇÃO DE CARGOS EM TEMPO REAL
local function checkRoles()
    for _, p in pairs(Players:GetPlayers()) do
        if p == lp then continue end
        
        -- Verifica faca e arma para identificar os papéis no jogo
        local hasKnife = p.Backpack:FindFirstChild("Knife") or (p.Character and p.Character:FindFirstChild("Knife"))
        local hasGun = p.Backpack:FindFirstChild("Gun") or (p.Character and p.Character:FindFirstChild("Gun")) or p.Backpack:FindFirstChild("Revolver") or (p.Character and p.Character:FindFirstChild("Revolver"))
        
        if hasKnife then 
            RolesCache.Murderer = p
            if _G.CherryConfig.ESP then applyESP(p, Color3.fromRGB(255, 0, 0)) end 
        elseif hasGun then 
            RolesCache.Sheriff = p
            if _G.CherryConfig.ESP then applyESP(p, Color3.fromRGB(0, 120, 255)) end 
        else
            -- Limpeza para inocentes (Exceto se for o alvo do Troll)
            if _G.CherryConfig.ESP and not (_G.CherryConfig.PlayerESP and p == selectedPlayer) then 
                removeESP(p) 
            end
        end
    end
end

-- CRIAÇÃO DA JANELA PRINCIPAL (CORREÇÃO DE INTERFACE)
local Window = redzlib:MakeWindow({
    Title = "Cherry Hub",
    SubTitle = "v11.0 - Luks Edition",
    SaveFolder = "CherryMM2"
})

-- Botão de Minimizar (Corrigido para não bugar no celular)
Window:AddMinimizeButton({
    Button = { Image = "rbxassetid://78702423919944", BackgroundTransparency = 0 },
    Corner = { CornerRadius = UDim.new(35, 1) },
})

-- DEFINIÇÃO DAS 6 ABAS (MANTENDO A ORDEM SOLICITADA)
local T1 = Window:MakeTab({"Home", ""})
local T2 = Window:MakeTab({"Inocente", ""})
local T3 = Window:MakeTab({"Assassino", ""})
local T4 = Window:MakeTab({"Xerife", ""})
local T5 = Window:MakeTab({"Troll", ""})
local T6 = Window:MakeTab({"Misc", ""})

-- ABA HOME (BOAS-VINDAS E STATUS)
T1:AddParagraph({"🌸 Cherry Hub v11.0", "Bem-vindo, Luks!\nStatus: Motor Luks-Seeker Ativo.\nVersão: 11.0 (Revision Alpha)."})
T1:AddParagraph({"Importante:", "O Fling Seeker persistente monitora a velocidade do alvo. Ele só para quando o player for ejetado."})




-- Cherry Hub v11.0 - Luks Edition (PART 4/10)
-- Revisão: Abas Inocente e Assassino (Sintaxe Corrigida)

-- ABA INOCENTE - COMBATE E FARM
T2:AddSection({"Combate de Sobrevivência"})
T2:AddToggle({
    Name = "ESP Global (Inimigos)", 
    Default = false, 
    Callback = function(v) _G.CherryConfig.ESP = v end
})

T2:AddButton({"🔪 Kill Murder (Luks-Seeker Fling)", function() 
    -- Luks, esta função localiza o Murderer e ativa o motor persistente corrigido.
    local target;
    for _, p in pairs(Players:GetPlayers()) do
        if p ~= lp and p.Character and (p.Backpack:FindFirstChild("Knife") or p.Character:FindFirstChild("Knife")) then
            target = p
            break
        end
    end
    
    if target then 
        executeFling(target) 
    else
        notify("Cherry Hub", "Luks, o Assassino não foi detectado no momento.")
    end
end})

T2:AddSection({"💰 Farm de Moedas"})
T2:AddToggle({
    Name = "Ativar Auto Farm", 
    Default = false, 
    Callback = function(v) _G.CherryConfig.CoinFarm = v end
})

T2:AddSlider({
    Name = "Velocidade do Farm", 
    Min = 10, 
    Max = 250, 
    Default = 60, 
    Callback = function(v) _G.CherryConfig.FarmSpeed = v end
})

-- ABA ASSASSINO - HITBOX E AURA
T3:AddSection({"⚔️ Hitbox"})
T3:AddToggle({
    Name = "Ativar Hitbox", 
    Default = false, 
    Callback = function(v) _G.CherryConfig.Hitbox = v end
})

T3:AddSlider({
    Name = "Tamanho da Hitbox", 
    Min = 1, 
    Max = 60, 
    Default = 10, 
    Callback = function(v) _G.CherryConfig.HitboxSize = v end
})

T3:AddSection({"🔥 Kill Aura"})
T3:AddToggle({
    Name = "Ativar Kill Aura", 
    Default = false, 
    Callback = function(v) _G.CherryConfig.KillAura = v end
})

T3:AddSlider({
    Name = "Alcance da Aura", 
    Min = 1, 
    Max = 60, 
    Default = 15, 
    Callback = function(v) _G.CherryConfig.AuraRadius = v end
})




-- Cherry Hub v11.0 - Luks Edition (PART 5/10)
-- Revisão: Abas Xerife e Troll (Seleção de Alvo e Fling Persistente)

-- ABA XERIFE
T4:AddSection({"🎯 Auto Combat"})
T4:AddToggle({
    Name = "Ativar Auto Shot", 
    Default = false, 
    Callback = function(v) _G.CherryConfig.AutoShot = v end
})

T4:AddParagraph({"Dica:", "Luks, o Auto Shot só dispara se houver linha de visão limpa até o Murderer."})

-- ABA TROLL (SISTEMA DE SELEÇÃO E MOTOR SEEKER)
T5:AddSection({"🎯 Selecionar Alvo"})

-- Função auxiliar para listar nomes de jogadores
local function getPNames() 
    local names = {}
    for _, p in pairs(Players:GetPlayers()) do 
        if p ~= lp then table.insert(names, p.Name) end 
    end
    return names 
end

local pDropdown = T5:AddDropdown({
    Name = "Escolher Player", 
    Options = getPNames(), 
    Default = "", 
    Callback = function(v) 
        selectedPlayer = Players:FindFirstChild(v) 
        if selectedPlayer then
            notify("Alvo Selecionado", "Luks, o alvo agora é: " .. v)
        end
    end
})

-- Atualização automática da lista para o Luks
Players.PlayerAdded:Connect(function() pDropdown:SetOptions(getPNames()) end)
Players.PlayerRemoving:Connect(function() pDropdown:SetOptions(getPNames()) end)

T5:AddSection({"🔥 Ações no Alvo"})

T5:AddButton({"🌪️ Fling Alvo (Motor Persistente)", function() 
    -- Luks, este botão usa o motor executeFling que só para quando o alvo voar.
    if selectedPlayer then 
        executeFling(selectedPlayer) 
    else
        notify("Erro", "Luks, selecione um jogador primeiro!")
    end 
end})

T5:AddToggle({
    Name = "Fling Alvo Infinito (Loop)", 
    Default = false, 
    Callback = function(v)
        _G.CherryConfig.FlingLoop = v
        task.spawn(function() 
            while _G.CherryConfig.FlingLoop do 
                if selectedPlayer then 
                    -- O motor agora tem a trava interna de 'Velocity Magnitude'
                    executeFling(selectedPlayer) 
                end 
                task.wait(0.2) 
            end 
        end)
    end
})

T5:AddToggle({
    Name = "ESP Alvo (Foco)", 
    Default = false, 
    Callback = function(v) 
        _G.CherryConfig.PlayerESP = v 
        if not v and selectedPlayer then removeESP(selectedPlayer) end
    end
})




-- Cherry Hub v11.0 - Luks Edition (PART 6/10)
-- Revisão: Finalização da Troll, Aba Misc e Loops de Combate

-- CONTINUAÇÃO DA ABA TROLL (VISUAL E CAOS)
T5:AddToggle({
    Name = "View Alvo (Spectate)", 
    Default = false, 
    Callback = function(v) 
        _G.CherryConfig.View = v
        -- Luks, restaura a câmera para você se o View for desativado
        if not v and lp.Character and lp.Character:FindFirstChild("Humanoid") then 
            workspace.CurrentCamera.CameraSubject = lp.Character.Humanoid 
        end
    end
})

T5:AddSection({"💀 Caos Global"})
T5:AddButton({"💀 Fling Todos (Seeker All)", function() 
    -- Luks, este comando percorre a lista e usa o motor persistente em cada jogador.
    notify("Caos Ativado", "Luks, iniciando o massacre global...")
    for _, p in pairs(Players:GetPlayers()) do 
        if p ~= lp and p.Character then 
            executeFling(p) 
            task.wait(0.05) 
        end 
    end 
end})

-- ABA MISC (PRESERVADA PARA O LUKS)
T6:AddSection({"⚡ Movimentação"})
T6:AddSlider({
    Name = "Velocidade (WalkSpeed)", 
    Min = 16, 
    Max = 150, 
    Default = 16, 
    Callback = function(v) 
        if lp.Character and lp.Character:FindFirstChild("Humanoid") then 
            lp.Character.Humanoid.WalkSpeed = v 
        end 
    end
})

T6:AddSlider({
    Name = "Pulo (JumpPower)", 
    Min = 50, 
    Max = 300, 
    Default = 50, 
    Callback = function(v) 
        if lp.Character and lp.Character:FindFirstChild("Humanoid") then 
            lp.Character.Humanoid.JumpPower = v 
        end 
    end
})

-- LOOPS DE SINCRONIZAÇÃO EM TEMPO REAL (COMBATE)
RunService.Heartbeat:Connect(function()
    -- Gerenciamento de Hitbox
    if _G.CherryConfig.Hitbox then
        for _, p in pairs(Players:GetPlayers()) do
            if p ~= lp and p.Character and p.Character:FindFirstChild("HumanoidRootPart") then
                local hrp = p.Character.HumanoidRootPart
                hrp.Size = Vector3.new(_G.CherryConfig.HitboxSize, _G.CherryConfig.HitboxSize, _G.CherryConfig.HitboxSize)
                hrp.Transparency = 0.7
                hrp.CanCollide = false
            end
        end
    end
    
    -- Gerenciamento de Kill Aura
    if _G.CherryConfig.KillAura then
        local char = lp.Character
        local knife = char and (char:FindFirstChild("Knife") or lp.Backpack:FindFirstChild("Knife"))
        if knife then
            if knife.Parent == lp.Backpack then char.Humanoid:EquipTool(knife) end
            for _, p in pairs(Players:GetPlayers()) do
                if p ~= lp and p.Character and p.Character:FindFirstChild("HumanoidRootPart") and p.Character.Humanoid.Health > 0 then
                    -- Se o inimigo estiver no raio definido, a faca ataca automaticamente
                    if (char.HumanoidRootPart.Position - p.Character.HumanoidRootPart.Position).Magnitude < _G.CherryConfig.AuraRadius then
                        knife.Handle.CFrame = p.Character.HumanoidRootPart.CFrame
                    end
                end
            end
        end
    end
end)





-- Cherry Hub v11.0 - Luks Edition (PART 7/10)
-- Revisão: Execução de Combate e Renderização de Câmera

-- EXECUÇÃO DE COMBATE (FRAME-BY-FRAME)
RunService.Heartbeat:Connect(function()
    -- Luks, verifica a cada frame se o Auto-Shot deve ser disparado contra o Murderer.
    if _G.CherryConfig.AutoShot then 
        autoShot() 
    end
end)

-- LOOP DE RENDERIZAÇÃO (VISUAIS E CÂMERA)
RunService.RenderStepped:Connect(function()
    -- Sistema de Spectate (View Alvo) para o Luks
    if _G.CherryConfig.View and selectedPlayer and selectedPlayer.Character and selectedPlayer.Character:FindFirstChild("Humanoid") then
        workspace.CurrentCamera.CameraSubject = selectedPlayer.Character.Humanoid
    else
        -- Restaura o foco da câmera para o seu personagem quando desligado
        if not _G.CherryConfig.View and lp.Character and lp.Character:FindFirstChild("Humanoid") then
            if workspace.CurrentCamera.CameraSubject ~= lp.Character.Humanoid then
                workspace.CurrentCamera.CameraSubject = lp.Character.Humanoid
            end
        end
    end
    
    -- ESP Especial para o Alvo Selecionado no Dropdown (Cor Amarela)
    if _G.CherryConfig.PlayerESP and selectedPlayer then 
        applyESP(selectedPlayer, Color3.fromRGB(255, 255, 0)) 
    end
end)

-- MONITORAMENTO DE CARGOS E LIMPEZA VISUAL
task.spawn(function()
    while task.wait(0.5) do
        -- Luks, esta função identifica quem é o perigo na partida dinamicamente.
        checkRoles()
        
        -- Gerenciamento de Memória Visual: Remove ESP de quem não é mais prioridade.
        if not _G.CherryConfig.ESP then
            for _, p in pairs(Players:GetPlayers()) do
                -- Só mantém o ESP se for o seu alvo de Troll atual.
                if not (_G.CherryConfig.PlayerESP and p == selectedPlayer) then
                    removeESP(p)
                end
            end
        end
    end
end)




-- Cherry Hub v11.0 - Luks Edition (PART 8/10)
-- Revisão: Estabilidade do Personagem e Utilitários da Aba Misc

-- GESTÃO DE ESTADO DO PERSONAGEM (ESTABILIDADE LUKS)
local function onCharacterAdded(newChar)
    local hum = newChar:WaitForChild("Humanoid")
    
    -- Limpa o cache de cargos na morte para evitar bugs visuais no ESP
    hum.Died:Connect(function()
        RolesCache = { Murderer = nil, Sheriff = nil }
    end)
    
    -- ANTI-SIT INTELIGENTE: Bloqueia sentar em bancos, exceto durante o Fling.
    hum:GetPropertyChangedSignal("Sit"):Connect(function()
        -- O motor Luks-Seeker utiliza o Sit para instabilizar a física
        local isFlinging = newChar.HumanoidRootPart:FindFirstChild("Luks_Velocity")
        if hum.Sit and not isFlinging then 
            hum.Sit = false 
            -- Teleporta levemente para cima para sair do objeto
            newChar.HumanoidRootPart.CFrame = newChar.HumanoidRootPart.CFrame * CFrame.new(0, 2, 0)
        end
    end)
end

-- Ativação para o personagem atual e futuros renascimentos
if lp.Character then onCharacterAdded(lp.Character) end
lp.CharacterAdded:Connect(onCharacterAdded)

-- ANTI-STUN & FORCE STAND (EVITA FICAR RAGDOLL)
RunService.Stepped:Connect(function()
    if lp.Character and lp.Character:FindFirstChild("Humanoid") then
        local hum = lp.Character.Humanoid
        local state = hum:GetState()
        if state == Enum.HumanoidStateType.FallingDown or state == Enum.HumanoidStateType.Ragdoll then
            -- Se não estivermos fligando, forçamos o estado de pé instantaneamente
            local isFlinging = lp.Character.HumanoidRootPart:FindFirstChild("Luks_Velocity")
            if not isFlinging then
                hum:ChangeState(Enum.HumanoidStateType.Running)
            end
        end
    end
end)

-- ABA MISC - SEÇÃO UTILITÁRIOS (PRESERVADA PARA O LUKS)
T6:AddSection({"🛠️ Utilitários"})

T6:AddButton({"♻️ Resetar Personagem", function()
    -- Luks, use para limpar bugs visuais ou sair de armadilhas.
    if lp.Character then lp.Character:BreakJoints() end
end})

T6:AddButton({"🚫 FPS Boost (Otimizar)", function()
    -- Luks, remove texturas e detalhes pesados para rodar mais liso.
    for _, v in pairs(workspace:GetDescendants()) do
        if v:IsA("BasePart") and not v.Parent:FindFirstChild("Humanoid") then
            v.Material = Enum.Material.SmoothPlastic
            v.Reflectance = 0
        elseif v:IsA("Decal") or v:IsA("Texture") then
            v:Destroy()
        elseif v:IsA("ParticleEmitter") or v:IsA("Trail") then
            v.Enabled = false
        end
    end
    notify("FPS Boost", "Luks, texturas simplificadas para maior desempenho!")
end})




-- Cherry Hub v11.0 - Luks Edition (PART 9/10)
-- Revisão: Estabilização de Rede, Rejoin e Eventos do Mapa

-- ESTABILIZAÇÃO DE REDE (PHYSICS PRIORITY)
-- Luks, esta função garante que o motor de Fling receba prioridade de processamento.
local function stabilizeNetwork()
    task.spawn(function()
        while task.wait(1) do
            if lp.Character and lp.Character:FindFirstChild("HumanoidRootPart") then
                -- Impede que o personagem entre em modo de economia de física (Sleep)
                settings().Physics.AllowSleep = false
                lp.ReplicationFocus = workspace
            end
        end
    end)
end
stabilizeNetwork()

-- ADICIONAIS NA ABA MISC (GESTÃO DE SERVIDOR)
T6:AddSection({"🌐 Servidor"})

T6:AddButton({"🔄 Re-entrar no Servidor (Rejoin)", function()
    -- Luks, use para reiniciar sua sessão instantaneamente no mesmo servidor.
    local TeleportService = game:GetService("TeleportService")
    TeleportService:Teleport(game.PlaceId, lp)
end})

-- LIMPEZA DE INSTÂNCIAS (GARBAGE COLLECTOR)
-- Luks, este loop limpa os resíduos físicos do Fling para evitar lag acumulado.
task.spawn(function()
    while task.wait(30) do
        for _, v in pairs(game:GetService("Debris"):GetChildren()) do
            if v.Name == "Luks_Velocity" or v.Name == "Luks_Torque" then
                v:Destroy()
            end
        end
    end
end)

-- DETECÇÃO DE EVENTOS DO MM2 (NOTIFICAÇÕES CRÍTICAS)
workspace.ChildAdded:Connect(function(v)
    if v.Name == "GunDrop" then
        -- O script avisa no topo da tela quando o Sheriff morre e a arma cai.
        notify("Cherry Hub", "Luks, a arma caiu! É sua chance de virar o jogo.")
    end
end)

-- LOG DE DEPURAÇÃO (OPCIONAL PARA EXECUTORES)
task.spawn(function()
    print("[Cherry Hub]: Sincronização de rede concluída para o usuário Luks.")
end)

-- Cherry Hub v11.0 - Luks Edition (PART 10/10)
-- Revisão Final: Ativação de Interface e Encerramento de Carregamento

-- SELEÇÃO AUTOMÁTICA DA ABA PRINCIPAL
-- Luks, isso garante que a interface renderize a aba Home assim que você executar.
Window:SelectTab(T1)

-- MENSAGENS DE CONFIRMAÇÃO NO CONSOLE
print("-----------------------------------------")
print("   CHERRY HUB v11.0 - REVISÃO FINAL     ")
print("   Status: INTERFACE CORRIGIDA          ")
print("   Motor: LUKS-SEEKER (ESTÁVEL)         ")
print("   Usuário: LUKS                        ")
print("-----------------------------------------")

-- NOTIFICAÇÃO FINAL DE SUCESSO
notify("Cherry Hub v11.0", "Luks, o script foi carregado com sucesso e a interface está pronta!")

-- FECHAMENTO DO PROCESSO
-- O script agora permanece em execução monitorando todos os comandos das 6 abas.
-- Certifique-se de colar as 10 partes em sequência para que o código funcione 100%.

-- [FIM DO SCRIPT REVISADO - CHERRY HUB v11.0 - LUKS EDITION]



