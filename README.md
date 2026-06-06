-- Cherry Hub v11.0 - Luks Edition (PART 1/100000)
-- Motor de Física: Luks-Seeker (Base v3.2 Improved)
-- Status: Fling Persistente + Velocidade GOD

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

-- MOTOR DE FLING "LUKS-SEEKER" (MELHORADO)
local function executeFling(targetPlayer)
    if not targetPlayer or not targetPlayer.Character then return end
    local myChar = lp.Character
    local myHRP = myChar and myChar:FindFirstChild("HumanoidRootPart")
    local myHum = myChar and myChar:FindFirstChild("Humanoid")
    local tHRP = targetPlayer.Character:FindFirstChild("HumanoidRootPart")
    
    if not myHRP or not tHRP or not myHum then return end
    
    local initialPos = myHRP.CFrame
    
    -- Ativação de Física Agressiva
    myHum.Sit = true 
    myHum:ChangeState(Enum.HumanoidStateType.Physics)
    
    -- Força de Velocidade Constante (BodyVelocity)
    local bv = Instance.new("BodyVelocity")
    bv.Name = "Luks_Velocity"
    bv.MaxForce = Vector3.new(1,1,1) * 9e18
    bv.Velocity = Vector3.new(9e8, 9e8, 9e8)
    bv.Parent = myHRP
    
    -- Torque de Rotação (O que faz o player voar longe)
    local bav = Instance.new("BodyAngularVelocity")
    bav.Name = "Luks_Torque"
    bav.MaxTorque = Vector3.new(1,1,1) * 9e18
    bav.AngularVelocity = Vector3.new(9e9, 9e9, 9e9)
    bav.Parent = myHRP
    
    -- Blindagem de Colisão
    for _, v in pairs(myChar:GetDescendants()) do
        if v:IsA("BasePart") then v.CanCollide = false; v.CanTouch = false end
    end
    
    local angle = 0
    -- LOOP PERSISTENTE: Só para quando o alvo for ejetado ou sair do jogo
    local connection
    connection = RunService.Heartbeat:Connect(function()
        if not targetPlayer.Parent or not targetPlayer.Character or not tHRP then 
            connection:Disconnect() 
            return 
        end
        
        angle = angle + 150 -- Rotação mais rápida que a v3.2
        
        -- Predição de Movimento Avançada (Pega o player mais rápido)
        local prediction = tHRP.Velocity * 0.12 
        local targetPos = tHRP.Position + prediction
        
        -- Gruda no alvo com CFrame dinâmico
        myHRP.CFrame = CFrame.new(targetPos) * CFrame.Angles(math.rad(angle), math.rad(angle), 0)
        
        -- Aplica velocidade de Assembly para garantir o "Nuke"
        myHRP.AssemblyLinearVelocity = Vector3.new(9e8, 9e8, 9e8)
    end)
    
    -- CONDIÇÃO DE PARADA: Só para se a velocidade do alvo for > 700 (Flingado com sucesso)
    -- Ou se passar de 4 segundos (Segurança para não travar o script)
    local timeout = tick()
    repeat 
        task.wait() 
    until (tHRP and tHRP.Velocity.Magnitude > 700) or (tick() - timeout > 4) or not targetPlayer.Parent
    
    -- LIMPEZA E RETORNO
    connection:Disconnect()
    bv:Destroy()
    bav:Destroy()
    
    myHRP.CFrame = initialPos
    myHum.Sit = false
    myHum:ChangeState(Enum.HumanoidStateType.Running)
    
    -- Estabilização de inércia
    for i = 1, 10 do
        myHRP.Velocity = Vector3.zero
        myHRP.RotVelocity = Vector3.zero
        RunService.Heartbeat:Wait()
    end
    
    for _, v in pairs(myChar:GetDescendants()) do
        if v:IsA("BasePart") then v.CanCollide = true; v.CanTouch = true end
    end
end



-- Cherry Hub v11.0 - Luks Edition (PART 2/10)

-- REVISÃO: AUTO SHOT (PRECISÃO DE VETORES)
-- Luks, esta função detecta o Murderer e dispara automaticamente se ele estiver no seu campo de visão.
local function autoShot()
    local murderer = RolesCache.Murderer
    if murderer and murderer.Character and murderer.Character:FindFirstChild("HumanoidRootPart") then
        local gun = lp.Character:FindFirstChild("Gun") or lp.Backpack:FindFirstChild("Gun") or lp.Character:FindFirstChild("Revolver") or lp.Backpack:FindFirstChild("Revolver")
        
        if gun then
            local targetHRP = murderer.Character.HumanoidRootPart
            local myHRP = lp.Character.HumanoidRootPart
            
            -- Raycast para verificar se não há paredes no caminho
            local direction = (targetHRP.Position - myHRP.Position).Unit
            local rayParams = RaycastParams.new()
            rayParams.FilterDescendantsInstances = {lp.Character, murderer.Character}
            rayParams.FilterType = Enum.RaycastFilterType.Exclude
            
            local rayResult = workspace:Raycast(myHRP.Position, direction * 300, rayParams)
            
            if not rayResult then
                if gun.Parent == lp.Backpack then lp.Character.Humanoid:EquipTool(gun) end
                
                local remote = gun:FindFirstChild("ShootGun", true) or game:GetService("ReplicatedStorage"):FindFirstChild("ShootGun", true)
                if remote then
                    -- Disparo com predição leve para compensar o ping
                    remote:FireServer(targetHRP.Position + (targetHRP.Velocity * 0.1), myHRP.Position, direction)
                    task.wait(0.3) 
                end
            end
        end
    end
end

-- REVISÃO: COIN FARM (TWEEN SISTEMA SEGURO)
local coinCollected = {}
local isTweening = false

local function findCoins()
    local coins = {}
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
    
    -- Luks, o loop espera o tween acabar ou o farm ser desativado
    repeat task.wait() until not isTweening or not _G.CherryConfig.CoinFarm
end

-- GERENCIADOR DE FARM (RODA EM SEGUNDO PLANO)
task.spawn(function()
    while true do
        task.wait(0.1)
        if _G.CherryConfig.CoinFarm and not isTweening and lp.Character and lp.Character:FindFirstChild("HumanoidRootPart") then
            local coins = findCoins()
            if #coins > 0 then
                -- Organiza por moeda mais próxima para economizar tempo
                table.sort(coins, function(a, b)
                    return (lp.Character.HumanoidRootPart.Position - a.Position).Magnitude < (lp.Character.HumanoidRootPart.Position - b.Position).Magnitude
                end)
                safeTeleport(coins[1])
            end
        end
    end
end)



-- Cherry Hub v11.0 - Luks Edition (PART 3/10)

-- SISTEMA DE ESP (HIGHLIGHT DE ALTA PERFORMANCE)
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

-- DETECÇÃO DE CARGOS (ROLES CACHE)
-- Luks, esta função mapeia quem é o Assassino e o Xerife para o ESP e para o Auto-Shot.
local function checkRoles()
    for _, p in pairs(Players:GetPlayers()) do
        if p == lp then continue end
        
        -- Verifica mochila e personagem simultaneamente
        local hasKnife = p.Backpack:FindFirstChild("Knife") or (p.Character and p.Character:FindFirstChild("Knife"))
        local hasGun = p.Backpack:FindFirstChild("Gun") or (p.Character and p.Character:FindFirstChild("Gun")) or p.Backpack:FindFirstChild("Revolver") or (p.Character and p.Character:FindFirstChild("Revolver"))
        
        if hasKnife then 
            RolesCache.Murderer = p
            if _G.CherryConfig.ESP then applyESP(p, Color3.fromRGB(255, 0, 0)) end -- Vermelho para o Murder
        elseif hasGun then 
            RolesCache.Sheriff = p
            if _G.CherryConfig.ESP then applyESP(p, Color3.fromRGB(0, 120, 255)) end -- Azul para o Sheriff
        else
            -- Limpeza de ESP para inocentes
            if _G.CherryConfig.ESP and not (_G.CherryConfig.PlayerESP and p == selectedPlayer) then 
                removeESP(p) 
            end
        end
    end
end

-- INICIALIZAÇÃO DA INTERFACE (REDZ LIBRARY)
local Window = redzlib:MakeWindow({
    Title = "Cherry Hub",
    SubTitle = "v11.0 - Luks Edition",
    SaveFolder = "CherryMM2"
})

-- Botão de Minimizar (Personalizado)
Window:AddMinimizeButton({
    Button = { Image = "rbxassetid://78702423919944", BackgroundTransparency = 0 },
    Corner = { CornerRadius = UDim.new(35, 1) },
})

-- Criação das Abas (T1 a T6)
local T1 = Window:MakeTab({"Home", ""})
local T2 = Window:MakeTab({"Inocente", ""})
local T3 = Window:MakeTab({"Assassino", ""})
local T4 = Window:MakeTab({"Xerife", ""})
local T5 = Window:MakeTab({"Troll", ""})
local T6 = Window:MakeTab({"Misc", ""})

-- ABA HOME (BOAS-VINDAS)
T1:AddParagraph({"🌸 Cherry Hub v11.0", "Luks Edition - O Fling que não desiste.\n\nMotor: Luks-Seeker (Ativo).\nFísica: Assembly Nuke.\nUsuário: Luks."})
T1:AddParagraph({"Destaque desta versão:", "O Fling agora só para quando o alvo atinge uma velocidade de ejeção fatal. Não há fuga."})

-- Cherry Hub v11.0 - Luks Edition (PART 4/10)

-- ABA INOCENTE - SEÇÃO COMBATE (LUKS-SEEKER INTEGRADO)
T2:AddSection({"Combate de Sobrevivência"})
T2:AddToggle({
    Name = "ESP Global (Murder/Sheriff)", 
    Default = false, 
    Callback = function(v) _G.CherryConfig.ESP = v end
})

T2:AddButton({"🔪 Kill Murder (Luks-Seeker Fling)", function() 
    -- Luks, esta função localiza o Murderer e ativa o fling persistente
    local target Murderer;
    for _, p in pairs(Players:GetPlayers()) do
        if p ~= lp and p.Character and (p.Backpack:FindFirstChild("Knife") or p.Character:FindFirstChild("Knife")) then
            target = p
            break
        end
    end
    if target then 
        executeFling(target) 
    else
        print("Assassino não encontrado ou ainda não puxou a faca.")
    end
end})

T2:AddButton({"🔫 Roubar Arma (Auto-Grab)", function() 
    -- Procura pela arma caída no chão e teleporta para coletar
    for _, v in pairs(workspace:GetChildren()) do
        if v.Name == "GunDrop" or v:FindFirstChild("GunDrop") then
            local handle = v:FindFirstChild("Handle") or v:IsA("BasePart") and v or v:FindFirstChildWhichIsA("BasePart", true)
            if handle then
                local oldPos = lp.Character.HumanoidRootPart.CFrame
                lp.Character.HumanoidRootPart.CFrame = handle.CFrame
                task.wait(0.2)
                lp.Character.HumanoidRootPart.CFrame = oldPos
                break
            end
        end
    end
end})

-- ABA INOCENTE - SEÇÃO FARM DE MOEDAS
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

-- ABA ASSASSINO - SEÇÃO HITBOX
T3:AddSection({"⚔️ Hitbox (Expansão)"})
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

-- ABA ASSASSINO - SEÇÃO KILL AURA
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

-- ABA XERIFE
T4:AddSection({"🎯 Auto Combat"})
T4:AddParagraph({"Aviso do Xerife:", "Luks, o Auto Shot usa Raycast para garantir que você não atire em paredes. Precisão total contra o Murderer."})

T4:AddToggle({
    Name = "Ativar Auto Shot", 
    Default = false, 
    Callback = function(v) _G.CherryConfig.AutoShot = v end
})

-- ABA TROLL (SISTEMA DE SELEÇÃO E ATAQUE PERSISTENTE)
local function getPNames() 
    local names = {}
    for _, p in pairs(Players:GetPlayers()) do 
        if p ~= lp then table.insert(names, p.Name) end 
    end
    return names 
end

local pDropdown = T5:AddDropdown({
    Name = "Escolher Player (Alvo)", 
    Options = getPNames(), 
    Default = "", 
    Callback = function(v) selectedPlayer = Players:FindFirstChild(v) end
})

-- Atualização dinâmica da lista para o Luks (Quem entra e quem sai)
Players.PlayerAdded:Connect(function() pDropdown:SetOptions(getPNames()) end)
Players.PlayerRemoving:Connect(function() pDropdown:SetOptions(getPNames()) end)

T5:AddSection({"🔥 Ações no Alvo"})

T5:AddButton({"🌪️ Fling Alvo (Luks-Seeker)", function() 
    -- Fling único mas persistente (só para quando o cara voar)
    if selectedPlayer then executeFling(selectedPlayer) end 
end})

T5:AddToggle({
    Name = "Fling Alvo Infinito (Loop)", 
    Default = false, 
    Callback = function(v)
        _G.CherryConfig.FlingLoop = v
        task.spawn(function() 
            while _G.CherryConfig.FlingLoop do 
                if selectedPlayer then 
                    -- Luks, o motor executeFling já tem a trava de "só parar quando flingar"
                    executeFling(selectedPlayer) 
                end 
                task.wait(0.1) -- Delay mínimo para estabilidade
            end 
        end)
    end
})

T5:AddToggle({
    Name = "ESP no Alvo (Target Focus)", 
    Default = false, 
    Callback = function(v) 
        _G.CherryConfig.PlayerESP = v
        if not v and selectedPlayer then removeESP(selectedPlayer) end 
    end
})



-- Cherry Hub v11.0 - Luks Edition (PART 6/10)

-- CONTINUAÇÃO DA ABA TROLL (CONTROLE VISUAL)
T5:AddToggle({
    Name = "View Alvo (Spectate)", 
    Default = false, 
    Callback = function(v) 
        _G.CherryConfig.View = v
        if not v and lp.Character and lp.Character:FindFirstChild("Humanoid") then 
            workspace.CurrentCamera.CameraSubject = lp.Character.Humanoid 
        end
    end
})

T5:AddSection({"💀 Caos Global (Seeker All)"})
T5:AddButton({"💀 Fling Todos os Jogadores", function() 
    -- Luks, este comando percorre a lista e usa o motor persistente em cada um.
    for _, p in pairs(Players:GetPlayers()) do 
        if p ~= lp and p.Character then 
            executeFling(p) 
            task.wait(0.05) -- Delay mínimo para não travar o seu jogo
        end 
    end 
end})

-- ABA MISC (PRESERVADA PARA O LUKS)
T6:AddSection({"⚡ Movimentação e Física"})
T6:AddParagraph({"Dica de Uso:", "Luks, use a velocidade para se aproximar do alvo mais rápido antes de ativar o Fling."})

T6:AddSlider({
    Name = "Velocidade (WalkSpeed)", 
    Min = 16, 
    Max = 150, 
    Default = 16, 
    Callback = function(v) if lp.Character then lp.Character.Humanoid.WalkSpeed = v end end
})

T6:AddSlider({
    Name = "Pulo (JumpPower)", 
    Min = 50, 
    Max = 300, 
    Default = 50, 
    Callback = function(v) if lp.Character then lp.Character.Humanoid.JumpPower = v end end
})

-- LOOPS DE SINCRONIZAÇÃO (HITBOX E KILL AURA)
RunService.Heartbeat:Connect(function()
    -- Gerenciamento de Hitbox em Tempo Real
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
    
    -- Gerenciamento de Kill Aura (Ghost Hit)
    if _G.CherryConfig.KillAura then
        local char = lp.Character
        local knife = char and (char:FindFirstChild("Knife") or lp.Backpack:FindFirstChild("Knife"))
        if knife then
            -- Auto-Equipar para o Luks
            if knife.Parent == lp.Backpack then char.Humanoid:EquipTool(knife) end
            for _, p in pairs(Players:GetPlayers()) do
                if p ~= lp and p.Character and p.Character:FindFirstChild("HumanoidRootPart") and p.Character.Humanoid.Health > 0 then
                    -- Se estiver no alcance, a faca "teleporta" o dano
                    if (char.HumanoidRootPart.Position - p.Character.HumanoidRootPart.Position).Magnitude < _G.CherryConfig.AuraRadius then
                        knife.Handle.CFrame = p.Character.HumanoidRootPart.CFrame
                    end
                end
            end
        end
    end
end)





-- Cherry Hub v11.0 - Luks Edition (PART 7/10)

-- EXECUÇÃO DE COMBATE EM TEMPO REAL
RunService.Heartbeat:Connect(function()
    -- Luks, este loop verifica a cada frame se o Auto-Shot deve ser disparado.
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
        -- Restaura o foco da câmera para o seu personagem
        if not _G.CherryConfig.View and lp.Character and lp.Character:FindFirstChild("Humanoid") then
            if workspace.CurrentCamera.CameraSubject ~= lp.Character.Humanoid then
                workspace.CurrentCamera.CameraSubject = lp.Character.Humanoid
            end
        end
    end
    
    -- ESP Especial para o Alvo Selecionado no Dropdown (Amarelo)
    if _G.CherryConfig.PlayerESP and selectedPlayer then 
        applyESP(selectedPlayer, Color3.fromRGB(255, 255, 0)) 
    end
end)

-- MONITORAMENTO DE CARGOS E LIMPEZA DE ESP
task.spawn(function()
    while task.wait(0.5) do
        -- Luks, esta função identifica quem é o Murder e o Sheriff dinamicamente.
        checkRoles()
        
        -- Limpeza de Memória Visual: Remove destaques de quem não é mais alvo ou saiu do jogo.
        if not _G.CherryConfig.ESP then
            for _, p in pairs(Players:GetPlayers()) do
                -- Mantém o destaque apenas se for o seu alvo de Troll.
                if not (_G.CherryConfig.PlayerESP and p == selectedPlayer) then
                    removeESP(p)
                end
            end
        end
    end
end)

-- Cherry Hub v11.0 - Luks Edition (PART 8/10)

-- GESTÃO DE ESTADO DO PERSONAGEM (ESTABILIDADE LUKS)
-- Luks, esta função garante que seu personagem renasça com todas as proteções ativas.
local function onCharacterAdded(newChar)
    local hum = newChar:WaitForChild("Humanoid")
    
    -- Limpa o cache de cargos na morte para evitar bugs de ESP
    hum.Died:Connect(function()
        RolesCache = { Murderer = nil, Sheriff = nil }
    end)
    
    -- ANTI-SIT INTELIGENTE: Impede que você sente em bancos, exceto durante o Fling.
    hum:GetPropertyChangedSignal("Sit"):Connect(function()
        -- O motor Luks-Seeker usa o Sit para instabilizar a física.
        -- Só bloqueamos o Sit se as forças de Fling NÃO estiverem ativas.
        local isFlinging = newChar.HumanoidRootPart:FindFirstChild("Luks_Velocity")
        if hum.Sit and not isFlinging then 
            hum.Sit = false 
            -- Teleporta levemente para cima para sair do banco/cadeira
            newChar.HumanoidRootPart.CFrame = newChar.HumanoidRootPart.CFrame * CFrame.new(0, 2, 0)
        end
    end)
end

-- Ativação para o personagem atual e futuros
if lp.Character then onCharacterAdded(lp.Character) end
lp.CharacterAdded:Connect(onCharacterAdded)

-- ANTI-STUN & FORCE STAND
-- Luks, isso impede que você caia ou fique "deitado" no chão (Ragdoll).
RunService.Stepped:Connect(function()
    if lp.Character and lp.Character:FindFirstChild("Humanoid") then
        local state = lp.Character.Humanoid:GetState()
        if state == Enum.HumanoidStateType.FallingDown or state == Enum.HumanoidStateType.Ragdoll then
            -- Se não estivermos fligando, forçamos o estado de pé
            local isFlinging = lp.Character.HumanoidRootPart:FindFirstChild("Luks_Velocity")
            if not isFlinging then
                lp.Character.Humanoid:ChangeState(Enum.HumanoidStateType.Running)
            end
        end
    end
end)

-- ABA MISC - SEÇÃO UTILITÁRIOS (PRESERVADA)
T6:AddSection({"🛠️ Utilitários de Sistema"})

T6:AddButton({"♻️ Resetar Personagem", function()
    -- Luks, use se o personagem bugar ou se você ficar preso em algum lugar.
    if lp.Character then lp.Character:BreakJoints() end
end})

T6:AddButton({"🚫 FPS Boost (Reduzir Lag)", function()
    -- Luks, esta função limpa texturas e partículas para o script rodar mais liso.
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
end})





-- Cherry Hub v11.0 - Luks Edition (PART 9/10)

-- ESTABILIZAÇÃO DE REDE (PHYSICS PRIORITY)
-- Luks, esta função garante que o motor de Fling receba prioridade total de processamento.
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
    -- Luks, usa este botão para reiniciar sua sessão instantaneamente.
    local TeleportService = game:GetService("TeleportService")
    TeleportService:Teleport(game.PlaceId, lp)
end})

-- LIMPEZA DE INSTÂNCIAS (GARBAGE COLLECTOR)
-- Luks, este loop limpa os resíduos do Fling (velocidade/torque) para evitar lag.
task.spawn(function()
    while task.wait(30) do
        for _, v in pairs(game:GetService("Debris"):GetChildren()) do
            if v.Name == "Luks_Velocity" or v.Name == "Luks_Torque" then
                v:Destroy()
            end
        end
    end
end)

-- SISTEMA DE NOTIFICAÇÃO (FEEDBACK PARA O LUKS)
local function notify(title, text)
    if redzlib.SetNotif then
        redzlib:SetNotif({
            Title = title,
            Description = text,
            Time = 5
        })
    else
        -- Fallback caso a biblioteca sofra atualização
        print("[" .. title .. "]: " .. text)
    end
end

-- Detecção de Eventos do MM2 (Avisos Críticos)
workspace.ChildAdded:Connect(function(v)
    if v.Name == "GunDrop" then
        -- Luks, o script avisa quando a arma está disponível para coleta.
        notify("Cherry Hub", "Atenção Luks: A arma caiu! Vá buscar.")
    end
end)



-- Cherry Hub v11.0 - Luks Edition (PART 10/10)

-- FINALIZAÇÃO E ATIVAÇÃO DA INTERFACE
-- Luks, esta linha garante que o script sempre abra na aba principal para você ver o status.
Window:SelectTab(T1)

-- MENSAGENS DE LOG (CONFIRMAÇÃO NO CONSOLE DO EXECUTOR)
print("-----------------------------------------")
print("   CHERRY HUB v11.0 - LUKS EDITION   ")
print("   Status: 100% CARREGADO           ")
print("   Usuário: LUKS                    ")
print("   Motor: LUKS-SEEKER (Ativo)       ")
print("   Fling: PERSISTENTE (MODO CARAPATO)")
print("-----------------------------------------")

-- Notificação Final de Inicialização para o Luks
notify("Cherry Hub v11.0", "Carregamento concluído! O Luks-Seeker está pronto para caçar.")

-- FECHAMENTO DO PROCESSO DE INICIALIZAÇÃO
-- O script agora entra em modo de escuta contínua de todos os loops de combate e troll.
-- Luks, certifique-se de que as 10 partes foram coladas sequencialmente para evitar erros.

-- [FIM DO SCRIPT CHERRY HUB v11.0 - LUKS EDITION]



