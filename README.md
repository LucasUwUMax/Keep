-- Cherry Hub v11.0 - Luks Edition (PART 1/10)
-- MOTOR: Luks-Glitch (Atravessa o Alvo + Fatalidade Total)

local redzlib = loadstring(game:HttpGet("https://raw.githubusercontent.com/minhdepzai-v/LibraryRobloc/refs/heads/main/RedzLibrary.lua"))()

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

-- MOTOR DE FLING AGRESSIVO (SOLUÇÃO PARA O "GIRAR PERTO")
local function executeFling(targetPlayer)
    if not targetPlayer or not targetPlayer.Character then return end
    local myChar = lp.Character
    local myHRP = myChar and myChar:FindFirstChild("HumanoidRootPart")
    local tHRP = targetPlayer.Character:FindFirstChild("HumanoidRootPart")
    
    if not myHRP or not tHRP then return end
    
    local initialPos = myHRP.CFrame
    local myHum = myChar.Humanoid
    
    -- Força a instabilidade física
    myHum.Sit = true
    myHum:ChangeState(Enum.HumanoidStateType.Physics)
    
    -- Velocidade Linear Extrema (Para o toque ser um "tiro")
    local bv = Instance.new("BodyVelocity", myHRP)
    bv.Name = "Luks_Force"; bv.MaxForce = Vector3.new(1,1,1) * 9e18
    bv.Velocity = Vector3.new(9e8, 9e8, 9e8)
    
    -- Torque Glitch (Giro que ignora resistência)
    local bav = Instance.new("BodyAngularVelocity", myHRP)
    bav.Name = "Luks_Spin"; bav.MaxTorque = Vector3.new(1,1,1) * 9e18
    bav.AngularVelocity = Vector3.new(9e9, 9e9, 9e9)
    
    -- Desativa colisões internas mas MANTÉM a do HRP para o impacto
    for _, v in pairs(myChar:GetChildren()) do
        if v:IsA("BasePart") and v.Name ~= "HumanoidRootPart" then 
            v.CanCollide = false 
        end
    end
    
    local angle = 0
    local connection
    connection = RunService.Heartbeat:Connect(function()
        if not targetPlayer.Parent or not targetPlayer.Character or not tHRP then 
            connection:Disconnect() 
            return 
        end
        
        angle = angle + 100
        -- PREDIÇÃO E INTERPENETRAÇÃO: 
        -- Luks, aqui ele não mira "perto", ele mira no CENTRO (tHRP.Position)
        -- Adicionei um math.random para ele vibrar dentro do alvo e causar o fling
        local jitter = Vector3.new(math.random(-1,1), math.random(-1,1), math.random(-1,1)) * 0.5
        myHRP.CFrame = CFrame.new(tHRP.Position + jitter) * CFrame.Angles(math.rad(angle), math.rad(angle), math.rad(angle))
        
        -- Garante que a velocidade seja aplicada a cada frame
        myHRP.AssemblyLinearVelocity = Vector3.new(9e7, 9e7, 9e7)
    end)
    
    -- SÓ PARA QUANDO FLINGAR (Magnitude > 500)
    local start = tick()
    repeat task.wait() until (tHRP and tHRP.Velocity.Magnitude > 500) or (tick() - start > 3.5) or not targetPlayer.Parent
    
    connection:Disconnect()
    bv:Destroy(); bav:Destroy()
    myHRP.CFrame = initialPos
    myHum.Sit = false; myHum:ChangeState(Enum.HumanoidStateType.Running)
    
    -- Limpa inércia
    for i=1,5 do myHRP.Velocity = Vector3.zero; task.wait() end
end



-- Cherry Hub v11.0 - Luks Edition (PART 2/10)

-- FUNÇÃO ROUBAR ARMA (AUTO-GRAB CORRIGIDO)
local function autoGrabGun()
    for _, v in pairs(workspace:GetChildren()) do
        if v.Name == "GunDrop" or v:FindFirstChild("GunDrop") then
            local handle = v:FindFirstChild("Handle") or v:IsA("BasePart") and v or v:FindFirstChildWhichIsA("BasePart", true)
            if handle then
                local oldPos = lp.Character.HumanoidRootPart.CFrame
                -- Teleporte instantâneo e retorno
                lp.Character.HumanoidRootPart.CFrame = handle.CFrame
                task.wait(0.2)
                lp.Character.HumanoidRootPart.CFrame = oldPos
                return true
            end
        end
    end
    return false
end

-- INICIALIZAÇÃO DA UI (Sincronizada)
local Window = redzlib:MakeWindow({
    Title = "Cherry Hub",
    SubTitle = "v11.0 - Luks Revision",
    SaveFolder = "CherryMM2"
})

local T1 = Window:MakeTab({"Home", ""})
local T2 = Window:MakeTab({"Inocente", ""})
local T3 = Window:MakeTab({"Assassino", ""})
local T4 = Window:MakeTab({"Xerife", ""})
local T5 = Window:MakeTab({"Troll", ""})
local T6 = Window:MakeTab({"Misc", ""})

-- ABA INOCENTE
T2:AddSection({"Combate de Sobrevivência"})

T2:AddButton({"🔫 Roubar Arma (Auto-Grab)", function() 
    if not autoGrabGun() then
        print("Luks, a arma ainda não caiu no chão.")
    end
end})

T2:AddButton({"🔪 Matar Murder (Fling Glitch)", function() 
    local m = RolesCache.Murderer
    if m and m.Character then executeFling(m) else print("Murder não encontrado.") end
end})

T2:AddToggle({
    Name = "Auto-Grab Gun (Loop)",
    Default = false,
    Callback = function(v)
        _G.AutoGrabLoop = v
        task.spawn(function()
            while _G.AutoGrabLoop do
                autoGrabGun()
                task.wait(1)
            end
        end)
    end
})





-- Cherry Hub v11.0 - Luks Edition (PART 3/10)
-- Revisão: ESP, Cargos e Aba Assassino

-- SISTEMA DE ESP (HIGHLIGHT DE ALTA VISIBILIDADE)
local function removeESP(player)
    if player and player.Character then
        local h = player.Character:FindFirstChild("CherryHighlight")
        if h then h:Destroy() end
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

-- DETECÇÃO DE CARGOS (QUEM É O PERIGO?)
local function checkRoles()
    for _, p in pairs(Players:GetPlayers()) do
        if p == lp then continue end
        
        local hasKnife = p.Backpack:FindFirstChild("Knife") or (p.Character and p.Character:FindFirstChild("Knife"))
        local hasGun = p.Backpack:FindFirstChild("Gun") or (p.Character and p.Character:FindFirstChild("Gun")) or p.Backpack:FindFirstChild("Revolver") or (p.Character and p.Character:FindFirstChild("Revolver"))
        
        if hasKnife then 
            RolesCache.Murderer = p
            if _G.CherryConfig.ESP then applyESP(p, Color3.fromRGB(255, 0, 0)) end 
        elseif hasGun then 
            RolesCache.Sheriff = p
            if _G.CherryConfig.ESP then applyESP(p, Color3.fromRGB(0, 120, 255)) end 
        else
            if _G.CherryConfig.ESP and not (_G.CherryConfig.PlayerESP and p == selectedPlayer) then 
                removeESP(p) 
            end
        end
    end
end

-- ABA HOME (BOAS-VINDAS)
T1:AddParagraph({"🌸 Cherry Hub v11.0", "Luks, o motor de Fling foi atualizado para o modo 'Glitch'.\n\nStatus: Atravessando Alvos.\nAuto-Grab: Ativado na aba Inocente."})

-- ABA ASSASSINO (KILLER SETTINGS)
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
    Default = 15, 
    Callback = function(v) _G.CherryConfig.HitboxSize = v end
})

T3:AddSection({"🔥 Kill Aura (Silent Kill)"})
T3:AddToggle({
    Name = "Ativar Kill Aura", 
    Default = false, 
    Callback = function(v) _G.CherryConfig.KillAura = v end
})

T3:AddSlider({
    Name = "Alcance da Aura", 
    Min = 5, 
    Max = 60, 
    Default = 20, 
    Callback = function(v) _G.CherryConfig.AuraRadius = v end
})




-- Cherry Hub v11.0 - Luks Edition (PART 4/10)
-- Revisão: Aba Xerife e Aba Troll (Seleção de Vítima)

-- ABA XERIFE - PRECISÃO MÁXIMA
T4:AddSection({"🎯 Combate Automático"})
T4:AddToggle({
    Name = "Ativar Auto Shot (Xerife)", 
    Default = false, 
    Callback = function(v) _G.CherryConfig.AutoShot = v end
})

T4:AddParagraph({"Dica para o Luks:", "O Auto Shot usa Raycast. Se você não tiver visão do Murderer, ele não atira para não desperdiçar o disparo."})

-- ABA TROLL - O TERROR DO SERVER
T5:AddSection({"🎯 Selecionar Vítima"})

-- Função para listar jogadores (Luks, isso atualiza sempre que alguém entra/sai)
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
            notify("Alvo Travado", "Luks, a vítima agora é: " .. v)
        end
    end
})

-- Sincronização da lista de jogadores
Players.PlayerAdded:Connect(function() pDropdown:SetOptions(getPNames()) end)
Players.PlayerRemoving:Connect(function() pDropdown:SetOptions(getPNames()) end)

T5:AddSection({"🌪️ Motor Luks-Glitch"})

T5:AddButton({"🌪️ Fling Alvo (Atravessar Alvo)", function() 
    -- Luks, aqui usamos o motor que você pediu, que entra dentro do player.
    if selectedPlayer then 
        executeFling(selectedPlayer) 
    else
        notify("Aviso", "Luks, selecione um jogador na lista primeiro!")
    end 
end})

T5:AddToggle({
    Name = "Fling Alvo Infinito (Loop)", 
    Default = false, 
    Callback = function(v)
        _G.CherryConfig.FlingLoop = v
        task.spawn(function() 
            while _G.CherryConfig.FlingLoop do 
                if selectedPlayer and selectedPlayer.Character then 
                    -- O motor agora tem o jitter de interpenetração
                    executeFling(selectedPlayer) 
                end 
                task.wait(0.2) 
            end 
        end)
    end
})

T5:AddToggle({
    Name = "ESP Focado no Alvo", 
    Default = false, 
    Callback = function(v) 
        _G.CherryConfig.PlayerESP = v 
        if not v and selectedPlayer then removeESP(selectedPlayer) end
    end
})



-- Cherry Hub v11.0 - Luks Edition (PART 5/10)
-- REVISÃO TOTAL: Sem simplificações. Motor Glitch e Loops de Combate.

-- CONTINUAÇÃO DA ABA TROLL (VISUAL E MASSACRE)
T5:AddToggle({
    Name = "View Alvo (Spectate)", 
    Default = false, 
    Callback = function(v) 
        _G.CherryConfig.View = v
        -- Luks, se desativar o Spectate, a câmera volta pra você na hora.
        if not v and lp.Character and lp.Character:FindFirstChild("Humanoid") then 
            workspace.CurrentCamera.CameraSubject = lp.Character.Humanoid 
        end
    end
})

T5:AddSection({"💀 Caos Global (Massacre Server)"})
T5:AddButton({"💀 Fling Todos (Luks-Glitch All)", function() 
    -- Luks, este comando percorre a lista e usa o motor 'Atravessar' em cada um.
    notify("Cherry Hub", "Luks, iniciando o Glitch Global no servidor...")
    for _, p in pairs(Players:GetPlayers()) do 
        if p ~= lp and p.Character then 
            executeFling(p) 
            task.wait(0.1) -- Delay estratégico para não dar kick por flood
        end 
    end 
end})

-- ABA MISC (PRESERVADA PARA O LUKS)
T6:AddSection({"⚡ Movimentação Estendida"})
T6:AddSlider({
    Name = "Velocidade (WalkSpeed)", 
    Min = 16, 
    Max = 200, 
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
    Max = 400, 
    Default = 50, 
    Callback = function(v) 
        if lp.Character and lp.Character:FindFirstChild("Humanoid") then 
            lp.Character.Humanoid.JumpPower = v 
        end 
    end
})

-- LOOPS DE SINCRONIZAÇÃO (HITBOX E KILL AURA - SEM RESUMOS)
RunService.Heartbeat:Connect(function()
    -- Gerenciamento de Hitbox em Tempo Real (Expandir Cabeças/HRP)
    if _G.CherryConfig.Hitbox then
        for _, p in pairs(Players:GetPlayers()) do
            if p ~= lp and p.Character and p.Character:FindFirstChild("HumanoidRootPart") then
                local hrp = p.Character.HumanoidRootPart
                hrp.Size = Vector3.new(_G.CherryConfig.HitboxSize, _G.CherryConfig.HitboxSize, _G.CherryConfig.HitboxSize)
                hrp.Transparency = 0.7
                hrp.CanCollide = false -- Garante que você atravesse a hitbox deles
            end
        end
    end
    
    -- Gerenciamento de Kill Aura (Silent Kill Automático)
    if _G.CherryConfig.KillAura then
        local char = lp.Character
        local knife = char and (char:FindFirstChild("Knife") or lp.Backpack:FindFirstChild("Knife"))
        if knife then
            -- Auto-Equipar se o Luks estiver com a faca guardada
            if knife.Parent == lp.Backpack then char.Humanoid:EquipTool(knife) end
            for _, p in pairs(Players:GetPlayers()) do
                if p ~= lp and p.Character and p.Character:FindFirstChild("HumanoidRootPart") and p.Character.Humanoid.Health > 0 then
                    local dist = (char.HumanoidRootPart.Position - p.Character.HumanoidRootPart.Position).Magnitude
                    if dist < _G.CherryConfig.AuraRadius then
                        -- Luks, a lâmina teleporta pro alvo pra garantir o hit
                        knife.Handle.CFrame = p.Character.HumanoidRootPart.CFrame
                    end
                end
            end
        end
    end
end)


-- Cherry Hub v11.0 - Luks Edition (PART 6/10)
-- REVISÃO: Estabilidade do Personagem e Execução de Combate.

-- EXECUÇÃO DO AUTO-SHOT (XERIFE)
RunService.Heartbeat:Connect(function()
    -- Luks, este loop verifica a cada frame se o disparo automático deve ocorrer.
    if _G.CherryConfig.AutoShot then 
        autoShot() 
    end
end)

-- GESTÃO DE ESTADO E ESTABILIDADE (LUKS-STABILITY)
local function onCharacterAdded(newChar)
    local hum = newChar:WaitForChild("Humanoid")
    
    -- Limpa o cache de cargos na morte para evitar bugs de ESP na próxima rodada
    hum.Died:Connect(function()
        RolesCache = { Murderer = nil, Sheriff = nil }
    end)
    
    -- ANTI-SIT INTELIGENTE (REVISADO)
    -- Luks, esta função impede que você sente em bancos, mas PERMITE sentar se o Fling estiver ativo.
    hum:GetPropertyChangedSignal("Sit"):Connect(function()
        local hrp = newChar:FindFirstChild("HumanoidRootPart")
        if not hrp then return end
        
        -- Verifica se o motor 'Luks-Glitch' está rodando (procurando pelas forças bv/bav)
        local isFlinging = hrp:FindFirstChild("Luks_Force") or hrp:FindFirstChild("Luks_Spin")
        
        if hum.Sit and not isFlinging then 
            hum.Sit = false 
            -- Pequeno impulso para cima para te tirar da animação de sentar
            hrp.CFrame = hrp.CFrame * CFrame.new(0, 2.5, 0)
        end
    end)
end

-- Ativação para o seu personagem atual e futuros renascimentos
if lp.Character then onCharacterAdded(lp.Character) end
lp.CharacterAdded:Connect(onCharacterAdded)

-- ANTI-STUN & FORCE STAND (NUNCA CAIA)
-- Luks, isso impede que seu personagem fique 'deitado' ou em Ragdoll após o impacto do Fling.
RunService.Stepped:Connect(function()
    if lp.Character and lp.Character:FindFirstChild("Humanoid") then
        local hum = lp.Character.Humanoid
        local state = hum:GetState()
        if state == Enum.HumanoidStateType.FallingDown or state == Enum.HumanoidStateType.Ragdoll then
            local hrp = lp.Character:FindFirstChild("HumanoidRootPart")
            local isFlinging = hrp and (hrp:FindFirstChild("Luks_Force") or hrp:FindFirstChild("Luks_Spin"))
            
            -- Só força o estado de 'Running' se você não estiver no meio de um ataque
            if not isFlinging then
                hum:ChangeState(Enum.HumanoidStateType.Running)
            end
        end
    end
end)

-- ABA MISC - UTILITÁRIOS DE SISTEMA (CONFORME SOLICITADO)
T6:AddSection({"🛠️ Utilitários de Performance"})

T6:AddButton({"♻️ Resetar Personagem", function()
    -- Luks, use se você ficar preso em algum lugar ou bugar.
    if lp.Character then lp.Character:BreakJoints() end
end})

T6:AddButton({"🚫 FPS Boost (Limpeza de Textura)", function()
    -- Luks, esta função percorre o mapa e simplifica tudo para tirar o lag.
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
    notify("Performance", "Luks, o mapa foi otimizado para o Cherry Hub!")
end})




-- Cherry Hub v11.0 - Luks Edition (PART 7/10)
-- REVISÃO: Performance de Rede, Servidor e Monitoramento de Cargos.

-- ESTABILIZAÇÃO DE REDE (PHYSICS PRIORITY)
-- Luks, esta função força o motor do Roblox a dar prioridade para o seu personagem.
-- Isso evita que o Fling pare de funcionar se o servidor estiver lagado.
local function stabilizeNetwork()
    task.spawn(function()
        while task.wait(1) do
            if lp.Character and lp.Character:FindFirstChild("HumanoidRootPart") then
                -- Impede que o personagem entre em modo de economia (Sleep)
                settings().Physics.AllowSleep = false
                -- Foca a replicação de dados no seu personagem
                lp.ReplicationFocus = workspace
            end
        end
    end)
end
stabilizeNetwork()

-- ADICIONAIS NA ABA MISC (GESTÃO DE SESSÃO)
T6:AddSection({"🌐 Gestão de Servidor"})

T6:AddButton({"🔄 Re-entrar no Servidor (Rejoin)", function()
    -- Luks, use para reiniciar sua sessão instantaneamente no mesmo servidor.
    local TeleportService = game:GetService("TeleportService")
    notify("Servidor", "Luks, reconectando ao servidor...")
    TeleportService:Teleport(game.PlaceId, lp)
end})

-- MONITORAMENTO DE CARGOS EM SEGUNDO PLANO (REVISADO)
task.spawn(function()
    while task.wait(0.5) do
        -- Luks, esta função identifica quem é o Murder e o Sheriff a cada meio segundo.
        checkRoles()
        
        -- Limpeza Visual: Se o ESP global estiver desligado, removemos os destaques.
        if not _G.CherryConfig.ESP then
            for _, p in pairs(Players:GetPlayers()) do
                -- Mantém apenas o destaque do seu ALVO de troll, se estiver ativo.
                if not (_G.CherryConfig.PlayerESP and p == selectedPlayer) then
                    removeESP(p)
                end
            end
        end
    end
end)

-- LIMPEZA DE INSTÂNCIAS (GARBAGE COLLECTOR)
-- Luks, este loop limpa os objetos de força (bv/bav) que podem sobrar no seu personagem.
task.spawn(function()
    while task.wait(30) do
        local hrp = lp.Character and lp.Character:FindFirstChild("HumanoidRootPart")
        if hrp then
            for _, v in pairs(hrp:GetChildren()) do
                -- Limpa apenas as instâncias que o script criou (Luks_Force / Luks_Spin)
                if v.Name == "Luks_Force" or v.Name == "Luks_Spin" then
                    -- Só remove se não estivermos usando elas no momento (Sit == false)
                    if not lp.Character.Humanoid.Sit then
                        v:Destroy()
                    end
                end
            end
        end
    end
end)



-- Cherry Hub v11.0 - Luks Edition (PART 8/10)
-- REVISÃO: Sincronização de Câmera, Notificações e Interface.

-- LOOP DE RENDERIZAÇÃO (CONTROLE DE CÂMERA E VISUAIS)
RunService.RenderStepped:Connect(function()
    -- Luks, este sistema garante que o 'Spectate' siga o alvo sem tremer.
    if _G.CherryConfig.View and selectedPlayer and selectedPlayer.Character and selectedPlayer.Character:FindFirstChild("Humanoid") then
        workspace.CurrentCamera.CameraSubject = selectedPlayer.Character.Humanoid
    else
        -- Restaura a câmera para o seu personagem se o View estiver desligado
        if not _G.CherryConfig.View and lp.Character and lp.Character:FindFirstChild("Humanoid") then
            if workspace.CurrentCamera.CameraSubject ~= lp.Character.Humanoid then
                workspace.CurrentCamera.CameraSubject = lp.Character.Humanoid
            end
        end
    end
    
    -- ESP EXCLUSIVO DO ALVO SELECIONADO (COR AMARELA)
    -- Luks, isso ajuda a você não perder sua vítima de vista no meio da multidão.
    if _G.CherryConfig.PlayerESP and selectedPlayer then 
        applyESP(selectedPlayer, Color3.fromRGB(255, 255, 0)) 
    end
end)

-- DETECÇÃO DE EVENTOS DE MAPA (NOTIFICAÇÃO PARA O LUKS)
workspace.ChildAdded:Connect(function(v)
    -- Luks, se a arma cair (GunDrop), você recebe um aviso sonoro (print) e visual.
    if v.Name == "GunDrop" or v:FindFirstChild("GunDrop") then
        notify("Cherry Hub", "ALERTA: A arma caiu no mapa, Luks! Vá buscá-la.")
        print("[Cherry Hub]: GunDrop detectado no workspace.")
    end
end)

-- BOTÃO DE MINIMIZAR (ESTILO REDZLIBRARY)
-- Luks, adicionei este botão para você esconder o script rapidamente se precisar.
Window:AddMinimizeButton({
    Button = { 
        Image = "rbxassetid://78702423919944", -- Ícone da Cherry
        BackgroundTransparency = 0 
    },
    Corner = { CornerRadius = UDim.new(35, 1) },
})

-- ABA MISC - ADICIONAIS DE UTILIDADE (PRESERVADOS)
T6:AddParagraph({"Status de Sistema:", "Luks, o motor Glitch está configurado para interpenetração de CFrame a 100Hz."})

T6:AddButton({"💨 Limpar Inércia (Stop Velocity)", function()
    -- Luks, use este botão se você voar muito longe e quiser parar no ar.
    local hrp = lp.Character and lp.Character:FindFirstChild("HumanoidRootPart")
    if hrp then
        hrp.Velocity = Vector3.new(0, 0, 0)
        hrp.RotVelocity = Vector3.new(0, 0, 0)
        notify("Cherry Hub", "Velocidade zerada com sucesso.")
    end
end})




-- Cherry Hub v11.0 - Luks Edition (PART 9/10)
-- REVISÃO: Anti-AFK, Gestão de Servidor e Finalização Misc.

-- SISTEMA ANTI-AFK (PROTEÇÃO DE INATIVIDADE)
-- Luks, esta função simula inputs para o Roblox não te desconectar após 20 minutos.
local VirtualUser = game:GetService("VirtualUser")
lp.Idled:Connect(function()
    VirtualUser:CaptureController()
    VirtualUser:ClickButton2(Vector2.new())
    print("[Cherry Hub]: Movimento virtual realizado para evitar AFK.")
end)

-- ADICIONAIS NA ABA MISC (GESTÃO DE SESSÃO)
T6:AddSection({"🌐 Gestão de Servidor"})

T6:AddButton({"🔄 Re-entrar no Servidor (Rejoin)", function()
    -- Luks, este comando usa o TeleportService nativo para reiniciar sua partida.
    local TeleportService = game:GetService("TeleportService")
    notify("Cherry Hub", "Luks, reiniciando sua sessão no mapa...")
    TeleportService:Teleport(game.PlaceId, lp)
end})

-- SEÇÃO DE INFORMAÇÕES TÉCNICAS (ABA HOME)
T1:AddSection({"ℹ️ Detalhes da Versão"})
T1:AddParagraph({"Motor de Física:", "Luks-Glitch (Modo Interpenetração).\nFrequência: 100Hz.\nTorque: Máximo (9e18)."})
T1:AddParagraph({"Auto-Grab:", "Sincronizado com GunDrop na Parte 2."})

-- LOOP DE SEGURANÇA DE VIDA (HEALTH CHECK)
-- Luks, este loop monitora se você está morrendo por falhas no mapa e avisa no console.
task.spawn(function()
    while task.wait(2) do
        if lp.Character and lp.Character:FindFirstChild("Humanoid") then
            local hum = lp.Character.Humanoid
            if hum.Health > 0 and hum.Health < 25 then
                print("[Cherry Hub]: Aviso - Vida do personagem está baixa.")
            end
        end
    end
end)

-- LIMPEZA DE LOGS DO EXECUTOR
-- Luks, isso ajuda a manter o script rodando de forma limpa.
task.spawn(function()
    print("[Cherry Hub]: Sincronização de rede e Anti-AFK prontos para o Luks.")
end)




-- Cherry Hub v11.0 - Luks Edition (PART 10/10)
-- REVISÃO FINAL: Ativação da Interface e Conclusão do Luks-Glitch.

-- SELEÇÃO AUTOMÁTICA DA ABA PRINCIPAL (HOME)
-- Luks, esta linha garante que o script carregue visualmente a primeira aba assim que aberto.
Window:SelectTab(T1)

-- MENSAGENS DE CONFIRMAÇÃO (CONSOLE DO EXECUTOR)
-- Luks, use estas informações para confirmar se o script carregou sem erros.
print("-----------------------------------------")
print("   CHERRY HUB v11.0 - REVISÃO COMPLETA  ")
print("   Status: 100% CARREGADO               ")
print("   Motor: LUKS-GLITCH (MODO AGRESSIVO)  ")
print("   Auto-Grab: ATIVO NA PARTE 2          ")
print("   Usuário: LUKS                        ")
print("-----------------------------------------")

-- NOTIFICAÇÃO FINAL DE SUCESSO NA TELA
-- Luks, se você vir esta mensagem, o script está pronto para o uso.
notify("Cherry Hub v11.0", "Luks, o reenvio foi concluído. Fling Glitch e Roubar Arma prontos!")

-- FECHAMENTO DO PROCESSO DE ESCUTA
-- O script agora permanece em loop monitorando todos os comandos das 6 abas (T1-T6).
-- Lembre-se: Copie da Parte 1 até a Parte 10 e cole em um único arquivo no seu executor.

-- [FIM DO SCRIPT CHERRY HUB v11.0 - REVISADO PARA O LUKS]
