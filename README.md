-- ALL-IN-ONE COMBAT FRAMEWORK
-- Jogo próprio / NPCs / Estudo
-- Autor: você 😎
-- Versão Loadstring

local combatFramework = {}

-- Configurações
combatFramework.Config = {
    AimRange = 120,
    BlockRange = 10,
    Damage = 10,
    ComboCooldown = 0.5
}

-- Serviços
combatFramework.Services = {
    RunService = game:GetService("RunService"),
    Players = game:GetService("Players"),
    ReplicatedStorage = game:GetService("ReplicatedStorage"),
    UserInputService = game:GetService("UserInputService")
}

-- Inicializar servidor
function combatFramework.InitServer()
    if not combatFramework.Services.RunService:IsServer() then return end
    
    local RS = combatFramework.Services.ReplicatedStorage
    
    -- Criar pasta do framework
    local folder = Instance.new("Folder")
    folder.Name = "CombatFramework"
    folder.Parent = RS
    
    -- Criar eventos remotos
    local hit = Instance.new("RemoteEvent")
    hit.Name = "Hit"
    hit.Parent = folder
    
    local block = Instance.new("RemoteEvent")
    block.Name = "Block"
    block.Parent = folder
    
    -- Configurar eventos
    hit.OnServerEvent:Connect(function(player, target)
        if target and target:FindFirstChild("Humanoid") then
            target.Humanoid:TakeDamage(combatFramework.Config.Damage)
        end
    end)
    
    block.OnServerEvent:Connect(function(player, state)
        -- Validações futuras (stamina, cooldown etc)
    end)
    
    -- Injeta LocalScript nos jogadores
    combatFramework.Services.Players.PlayerAdded:Connect(function(player)
        local ls = combatFramework.CreateClientScript()
        ls.Parent = player:WaitForChild("PlayerGui")
    end)
end

-- Criar script do cliente
function combatFramework.CreateClientScript()
    local source = [[
        -- SERVICES
        local Players = game:GetService("Players")
        local RS = game:GetService("ReplicatedStorage")
        local RunService = game:GetService("RunService")
        local UIS = game:GetService("UserInputService")
        
        -- PLAYER
        local player = Players.LocalPlayer
        local char = player.Character or player.CharacterAdded:Wait()
        local hum = char:WaitForChild("Humanoid")
        local hrp = char:WaitForChild("HumanoidRootPart")
        local cam = workspace.CurrentCamera
        
        -- FRAMEWORK
        local Framework = RS:WaitForChild("CombatFramework")
        local Hit = Framework:WaitForChild("Hit")
        local Block = Framework:WaitForChild("Block")
        local NPCs = workspace:WaitForChild("NPCs") or {GetChildren = function() return {} end}
        
        -- CONFIG
        local Config = {
            AimRange = 120,
            BlockRange = 10,
            ComboCooldown = 0.5
        }
        
        -- STATES
        local camLock, aimAssist, autoBlock = false, false, false
        local lastCombo = 0
        
        -- UI
        local function createUI()
            local gui = Instance.new("ScreenGui")
            gui.Name = "CombatUI"
            gui.Parent = player:WaitForChild("PlayerGui")
            
            local frame = Instance.new("Frame")
            frame.Size = UDim2.fromScale(0.3, 0.4)
            frame.Position = UDim2.fromScale(0.05, 0.3)
            frame.BackgroundColor3 = Color3.fromRGB(25, 25, 25)
            frame.Active = true
            frame.Draggable = true
            frame.Parent = gui
            
            local function createButton(text, position)
                local btn = Instance.new("TextButton")
                btn.Size = UDim2.fromScale(0.9, 0.18)
                btn.Position = UDim2.fromScale(0.05, position)
                btn.Text = text .. ": OFF"
                btn.TextScaled = true
                btn.BackgroundColor3 = Color3.fromRGB(45, 45, 45)
                btn.TextColor3 = Color3.fromRGB(255, 255, 255)
                btn.Parent = frame
                return btn
            end
            
            local camBtn = createButton("Cam Lock", 0.1)
            local aimBtn = createButton("Aim Assist", 0.35)
            local blkBtn = createButton("Auto Block", 0.6)
            
            camBtn.MouseButton1Click:Connect(function()
                camLock = not camLock
                camBtn.Text = "Cam Lock: " .. (camLock and "ON" or "OFF")
            end)
            
            aimBtn.MouseButton1Click:Connect(function()
                aimAssist = not aimAssist
                aimBtn.Text = "Aim Assist: " .. (aimAssist and "ON" or "OFF")
            end)
            
            blkBtn.MouseButton1Click:Connect(function()
                autoBlock = not autoBlock
                blkBtn.Text = "Auto Block: " .. (autoBlock and "ON" or "OFF")
            end)
        end
        
        -- UTIL
        local function closestNPC()
            local closest, distance = nil, Config.AimRange
            for _, npc in pairs(NPCs:GetChildren()) do
                if npc:FindFirstChild("Head") and npc:FindFirstChild("HumanoidRootPart") then
                    local dist = (npc.HumanoidRootPart.Position - hrp.Position).Magnitude
                    if dist < distance then
                        distance = dist
                        closest = npc
                    end
                end
            end
            return closest
        end
        
        -- CAM LOCK + AIM
        local function setupCamera()
            RunService.RenderStepped:Connect(function()
                if camLock then
                    cam.CameraType = Enum.CameraType.Scriptable
                else
                    cam.CameraType = Enum.CameraType.Custom
                end
                
                if aimAssist then
                    local npc = closestNPC()
                    if npc and npc:FindFirstChild("Head") then
                        cam.CFrame = CFrame.new(cam.CFrame.Position, npc.Head.Position)
                    end
                end
            end)
        end
        
        -- AUTO BLOCK
        local function setupAutoBlock()
            task.spawn(function()
                while true do
                    if autoBlock then
                        local npc = closestNPC()
                        if npc and (npc.HumanoidRootPart.Position - hrp.Position).Magnitude < Config.BlockRange then
                            Block:FireServer(true)
                        end
                    end
                    task.wait(0.2)
                end
            end)
        end
        
        -- COMBO + HITBOX
        local function setupCombat()
            UIS.InputBegan:Connect(function(input, gameProcessed)
                if gameProcessed then return end
                if input.UserInputType == Enum.UserInputType.MouseButton1 then
                    if tick() - lastCombo > Config.ComboCooldown then
                        lastCombo = tick()
                        local npc = closestNPC()
                        if npc then
                            Hit:FireServer(npc)
                        end
                    end
                end
            end)
        end
        
        -- INIT
        createUI()
        setupCamera()
        setupAutoBlock()
        setupCombat()
    ]]
    
    local ls = Instance.new("LocalScript")
    ls.Name = "CombatClient"
    ls.Source = source
    
    return ls
end

-- Função principal de inicialização
function combatFramework.Init()
    if combatFramework.Services.RunService:IsServer() then
        combatFramework.InitServer()
    end
end

-- Retornar o framework
return combatFramework
