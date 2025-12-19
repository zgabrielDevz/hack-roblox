-- Painel de utilitários (educacional). Use apenas em lugares de teste.
local Players = game:GetService("Players")
local UserInputService = game:GetService("UserInputService")
local RunService = game:GetService("RunService")
local LocalPlayer = Players.LocalPlayer

-- GUI
local screenGui = Instance.new("ScreenGui")
screenGui.Name = "UtilityPanel"
screenGui.ResetOnSpawn = false
screenGui.Parent = LocalPlayer:WaitForChild("PlayerGui")

local frame = Instance.new("Frame")
frame.Size = UDim2.new(0, 220, 0, 200)
frame.Position = UDim2.new(0, 10, 0, 50)
frame.BackgroundTransparency = 0.2
frame.BackgroundColor3 = Color3.fromRGB(30,30,30)
frame.BorderSizePixel = 0
frame.Parent = screenGui

local function makeButton(text, y)
    local b = Instance.new("TextButton")
    b.Size = UDim2.new(0,200,0,30)
    b.Position = UDim2.new(0,10,0,y)
    b.Text = text
    b.Font = Enum.Font.SourceSans
    b.TextSize = 18
    b.Parent = frame
    return b
end

local espToggle = makeButton("ESP: OFF", 10)
local flyToggle = makeButton("Fly: OFF", 50)
local infJumpToggle = makeButton("Infinite Jump: OFF", 90)
local offAll = makeButton("Desligar tudo", 130)

-- Estado
local espEnabled = false
local flyEnabled = false
local infJumpEnabled = false

-- ESP implementation (billboard with name + distance)
local espFolder = Instance.new("Folder", workspace)
espFolder.Name = "ESP_Utility_Folder"

local function createBillboardForCharacter(char)
    if not char then return end
    local head = char:FindFirstChild("Head")
    if not head then return end
    local existing = head:FindFirstChild("ESP_Billboard")
    if existing then return existing end
    local bb = Instance.new("BillboardGui")
    bb.Name = "ESP_Billboard"
    bb.Size = UDim2.new(0,150,0,40)
    bb.StudsOffset = Vector3.new(0, 1.5, 0)
    bb.AlwaysOnTop = true
    bb.Parent = head

    local label = Instance.new("TextLabel")
    label.Size = UDim2.new(1,0,1,0)
    label.BackgroundTransparency = 1
    label.TextColor3 = Color3.new(1,1,1)
    label.TextStrokeTransparency = 0.5
    label.Font = Enum.Font.SourceSansBold
    label.TextSize = 14
    label.Text = "Player"
    label.Parent = bb
    return bb
end

local function removeESPForCharacter(char)
    if char and char:FindFirstChild("Head") then
        local bb = char.Head:FindFirstChild("ESP_Billboard")
        if bb then bb:Destroy() end
    end
end

local function updateESP()
    for _, pl in pairs(Players:GetPlayers()) do
        if pl.Character and pl.Character:FindFirstChild("Head") and pl ~= LocalPlayer then
            local bb = pl.Character.Head:FindFirstChild("ESP_Billboard")
            if espEnabled then
                if not bb then bb = createBillboardForCharacter(pl.Character) end
                local dist = math.floor((LocalPlayer.Character and LocalPlayer.Character:FindFirstChild("HumanoidRootPart") and (pl.Character.PrimaryPart.Position - LocalPlayer.Character.PrimaryPart.Position).Magnitude) or 0)
                bb.TextLabel.Text = pl.Name.." ("..tostring(dist).."m)"
            else
                removeESPForCharacter(pl.Character)
            end
        else
            -- tentar remover se existir
            if pl.Character then removeESPForCharacter(pl.Character) end
        end
    end
end

-- ligar/desligar ESP ao entrar/sair
Players.PlayerAdded:Connect(function(pl)
    pl.CharacterAdded:Connect(function(char)
        if espEnabled then
            wait(0.5) -- espera o Head existir
            if espEnabled then createBillboardForCharacter(char) end
        end
    end)
end)

Players.PlayerRemoving:Connect(function(pl)
    if pl.Character then removeESPForCharacter(pl.Character) end
end)

-- Fly implementation (local, usando BodyVelocity)
local flyForce
local flySpeed = 100
local flyUpDownSpeed = 70
local flyControl = {
    forward = 0,
    right = 0,
    up = 0,
}

local function enableFly()
    if not LocalPlayer.Character or not LocalPlayer.Character:FindFirstChild("HumanoidRootPart") then return end
    local hrp = LocalPlayer.Character.HumanoidRootPart
    flyForce = Instance.new("BodyVelocity")
    flyForce.MaxForce = Vector3.new(1e5,1e5,1e5)
    flyForce.Velocity = Vector3.new(0,0,0)
    flyForce.P = 1e4
    flyForce.Parent = hrp
end

local function disableFly()
    if flyForce then
        flyForce:Destroy()
        flyForce = nil
    end
end

-- atualiza velocidade de voo cada frame
RunService.RenderStepped:Connect(function(delta)
    if flyEnabled and flyForce and LocalPlayer.Character and LocalPlayer.Character:FindFirstChild("HumanoidRootPart") and LocalPlayer.Character:FindFirstChildOfClass("Humanoid") then
        local hrp = LocalPlayer.Character.HumanoidRootPart
        local cam = workspace.CurrentCamera
        local forward = cam.CFrame.LookVector * flyControl.forward
        local right = cam.CFrame.RightVector * flyControl.right
        local up = Vector3.new(0, flyControl.up, 0)
        local move = (forward + right).Unit
        if move ~= move then move = Vector3.new(0,0,0) end -- NaN check
        local vel = (move * flySpeed) + up * flyUpDownSpeed
        flyForce.Velocity = vel
    end
    if espEnabled then updateESP() end
end)

-- Inputs para controlar voo
UserInputService.InputBegan:Connect(function(input, gameProcessed)
    if gameProcessed then return end
    if input.KeyCode == Enum.KeyCode.W then flyControl.forward = 1 end
    if input.KeyCode == Enum.KeyCode.S then flyControl.forward = -1 end
    if input.KeyCode == Enum.KeyCode.A then flyControl.right = -1 end
    if input.KeyCode == Enum.KeyCode.D then flyControl.right = 1 end
    if input.KeyCode == Enum.KeyCode.E then flyControl.up = 1 end -- sobe
    if input.KeyCode == Enum.KeyCode.Q then flyControl.up = -1 end -- desce
end)

UserInputService.InputEnded:Connect(function(input)
    if input.KeyCode == Enum.KeyCode.W or input.KeyCode == Enum.KeyCode.S then flyControl.forward = 0 end
    if input.KeyCode == Enum.KeyCode.A or input.KeyCode == Enum.KeyCode.D then flyControl.right = 0 end
    if input.KeyCode == Enum.KeyCode.E or input.KeyCode == Enum.KeyCode.Q then flyControl.up = 0 end
end)

-- Infinite jump
local function onJumpRequest()
    if not infJumpEnabled then return end
    local char = LocalPlayer.Character
    if not char then return end
    local humanoid = char:FindFirstChildOfClass("Humanoid")
    if humanoid then
        humanoid:ChangeState(Enum.HumanoidStateType.Jumping)
    end
end

UserInputService.JumpRequest:Connect(onJumpRequest)

-- Botões
espToggle.MouseButton1Click:Connect(function()
    espEnabled = not espEnabled
    espToggle.Text = "ESP: "..(espEnabled and "ON" or "OFF")
    if not espEnabled then
        -- remover todos billboards
        for _, pl in pairs(Players:GetPlayers()) do
            if pl.Character then removeESPForCharacter(pl.Character) end
        end
    end
end)

flyToggle.MouseButton1Click:Connect(function()
    flyEnabled = not flyEnabled
    flyToggle.Text = "Fly: "..(flyEnabled and "ON" or "OFF")
    if flyEnabled then
        enableFly()
    else
        disableFly()
    end
end)

infJumpToggle.MouseButton1Click:Connect(function()
    infJumpEnabled = not infJumpEnabled
    infJumpToggle.Text = "Infinite Jump: "..(infJumpEnabled and "ON" or "OFF")
end)

offAll.MouseButton1Click:Connect(function()
    espEnabled = false
    flyEnabled = false
    infJumpEnabled = false
    espToggle.Text = "ESP: OFF"
    flyToggle.Text = "Fly: OFF"
    infJumpToggle.Text = "Infinite Jump: OFF"
    disableFly()
    for _, pl in pairs(Players:GetPlayers()) do
        if pl.Character then removeESPForCharacter(pl.Character) end
    end
end)

-- Lidar com respawn: garantir que fly seja reiniciado/limpo
LocalPlayer.CharacterAdded:Connect(function(char)
    wait(1)
    disableFly()
    if espEnabled then
        wait(0.5)
        updateESP()
    end
end)

-- Aviso pequeno no GUI
local note = Instance.new("TextLabel")
note.Size = UDim2.new(0,200,0,20)
note.Position = UDim2.new(0,10,0,170)
note.BackgroundTransparency = 1
note.TextColor3 = Color3.fromRGB(200,200,200)
note.Font = Enum.Font.SourceSansItalic
note.TextSize = 12
note.Text = "Uso educativo — apenas em testes."
note.Parent = frame
