local Players = game:GetService("Players")
local UserInputService = game:GetService("UserInputService")
local RunService = game:GetService("RunService")
local TweenService = game:GetService("TweenService")
local localPlayer = Players.LocalPlayer
local camera = workspace.CurrentCamera

-- Create the main GUI
local screenGui = Instance.new("ScreenGui")
screenGui.Name = "PlayerViewerGUI"
screenGui.Parent = localPlayer:WaitForChild("PlayerGui")

-- Main frame
local mainFrame = Instance.new("Frame")
mainFrame.Size = UDim2.new(0, 300, 0, 400)
mainFrame.Position = UDim2.new(0, 50, 0, 50)
mainFrame.BackgroundColor3 = Color3.fromRGB(40, 40, 40)
mainFrame.BorderSizePixel = 0
mainFrame.Active = true
mainFrame.Draggable = true
mainFrame.Parent = screenGui

-- Add shadow for better visibility
local shadow = Instance.new("Frame")
shadow.Size = UDim2.new(1, 6, 1, 6)
shadow.Position = UDim2.new(0, -3, 0, -3)
shadow.BackgroundColor3 = Color3.fromRGB(0, 0, 0)
shadow.BackgroundTransparency = 0.7
shadow.BorderSizePixel = 0
shadow.ZIndex = -1
shadow.Parent = mainFrame

-- Title bar (also serves as drag area)
local titleBar = Instance.new("Frame")
titleBar.Size = UDim2.new(1, 0, 0, 30)
titleBar.Position = UDim2.new(0, 0, 0, 0)
titleBar.BackgroundColor3 = Color3.fromRGB(30, 30, 30)
titleBar.Parent = mainFrame

-- Title text
local title = Instance.new("TextLabel")
title.Size = UDim2.new(1, -60, 1, 0)
title.Position = UDim2.new(0, 5, 0, 0)
title.BackgroundTransparency = 1
title.Text = "👀 Player Viewer"
title.TextColor3 = Color3.fromRGB(255, 255, 255)
title.Font = Enum.Font.SourceSansBold
title.TextSize = 18
title.TextXAlignment = Enum.TextXAlignment.Left
title.Parent = titleBar

-- Minimize button
local minimizeButton = Instance.new("TextButton")
minimizeButton.Size = UDim2.new(0, 25, 0, 25)
minimizeButton.Position = UDim2.new(1, -55, 0.5, -12)
minimizeButton.BackgroundColor3 = Color3.fromRGB(100, 100, 100)
minimizeButton.Text = "_"
minimizeButton.TextColor3 = Color3.fromRGB(255, 255, 255)
minimizeButton.Font = Enum.Font.SourceSansBold
minimizeButton.TextSize = 18
minimizeButton.Parent = titleBar

-- Close button
local closeButton = Instance.new("TextButton")
closeButton.Size = UDim2.new(0, 25, 0, 25)
closeButton.Position = UDim2.new(1, -25, 0.5, -12)
closeButton.BackgroundColor3 = Color3.fromRGB(200, 50, 50)
closeButton.Text = "X"
closeButton.TextColor3 = Color3.fromRGB(255, 255, 255)
closeButton.Font = Enum.Font.SourceSansBold
closeButton.TextSize = 18
closeButton.Parent = titleBar

-- Player list container (will be hidden when minimized)
local contentFrame = Instance.new("Frame")
contentFrame.Size = UDim2.new(1, 0, 1, -30)
contentFrame.Position = UDim2.new(0, 0, 0, 30)
contentFrame.BackgroundTransparency = 1
contentFrame.ClipsDescendants = true
contentFrame.Parent = mainFrame

-- Player list
local playerList = Instance.new("ScrollingFrame")
playerList.Size = UDim2.new(1, -10, 1, -90)
playerList.Position = UDim2.new(0, 5, 0, 5)
playerList.BackgroundTransparency = 1
playerList.ScrollBarThickness = 5
playerList.CanvasSize = UDim2.new(0, 0, 0, 0)
playerList.Parent = contentFrame

-- View button
local viewButton = Instance.new("TextButton")
viewButton.Size = UDim2.new(0.45, 0, 0, 40)
viewButton.Position = UDim2.new(0.025, 0, 1, -50)
viewButton.BackgroundColor3 = Color3.fromRGB(60, 120, 60)
viewButton.Text = "🔍 View"
viewButton.TextColor3 = Color3.fromRGB(255, 255, 255)
viewButton.Font = Enum.Font.SourceSansBold
viewButton.TextSize = 18
viewButton.Parent = contentFrame

-- Unview button
local unviewButton = Instance.new("TextButton")
unviewButton.Size = UDim2.new(0.45, 0, 0, 40)
unviewButton.Position = UDim2.new(0.525, 0, 1, -50)
unviewButton.BackgroundColor3 = Color3.fromRGB(120, 60, 60)
unviewButton.Text = "❌ Unview"
unviewButton.TextColor3 = Color3.fromRGB(255, 255, 255)
unviewButton.Font = Enum.Font.SourceSansBold
unviewButton.TextSize = 18
unviewButton.Parent = contentFrame

-- Variables
local selectedPlayer = nil
local renderConnection = nil
local cameraOffset = Vector3.new(0, 2, 5)
local cameraAngleX = 0
local cameraAngleY = 0
local isViewing = false
local isMinimized = false

-- Function to update player list
local function updatePlayerList()
    playerList:ClearAllChildren()
    
    local playerCount = 0
    for _, otherPlayer in ipairs(Players:GetPlayers()) do
        if otherPlayer ~= localPlayer then
            playerCount = playerCount + 1
            local playerButton = Instance.new("TextButton")
            playerButton.Size = UDim2.new(1, -10, 0, 30)
            playerButton.Position = UDim2.new(0, 5, 0, (playerCount-1) * 35)
            playerButton.BackgroundColor3 = Color3.fromRGB(60, 60, 60)
            playerButton.Text = "👤 " .. otherPlayer.Name
            playerButton.TextColor3 = Color3.fromRGB(255, 255, 255)
            playerButton.Font = Enum.Font.SourceSans
            playerButton.TextSize = 16
            playerButton.Parent = playerList
            
            playerButton.MouseButton1Click:Connect(function()
                -- Deselect previous
                for _, child in ipairs(playerList:GetChildren()) do
                    if child:IsA("TextButton") then
                        child.BackgroundColor3 = Color3.fromRGB(60, 60, 60)
                    end
                end
                
                -- Select new
                playerButton.BackgroundColor3 = Color3.fromRGB(80, 80, 120)
                selectedPlayer = otherPlayer
            end)
        end
    end
    
    playerList.CanvasSize = UDim2.new(0, 0, 0, playerCount * 35)
end

-- Function to view another player with rotatable camera
local function viewPlayer(targetPlayer)
    -- Stop any current viewing
    if renderConnection then
        renderConnection:Disconnect()
        renderConnection = nil
    end
    
    -- Check if player exists and has a character
    if targetPlayer and targetPlayer.Character then
        local humanoid = targetPlayer.Character:FindFirstChildOfClass("Humanoid")
        if humanoid then
            -- Set camera to scriptable
            camera.CameraType = Enum.CameraType.Scriptable
            isViewing = true
            
            -- Reset camera angles
            cameraAngleX = 0
            cameraAngleY = 0
            
            -- Update view every frame
            renderConnection = RunService.RenderStepped:Connect(function()
                if targetPlayer.Character and humanoid.RootPart then
                    -- Calculate camera position based on angles
                    local offset = CFrame.new(cameraOffset)
                    offset = offset * CFrame.Angles(0, math.rad(cameraAngleY), 0)
                    offset = offset * CFrame.Angles(math.rad(cameraAngleX), 0, 0)
                    
                    local finalOffset = offset.Position
                    camera.CFrame = CFrame.new(humanoid.RootPart.Position + finalOffset, humanoid.RootPart.Position)
                else
                    -- If player no longer has character, stop viewing
                    if renderConnection then
                        renderConnection:Disconnect()
                        renderConnection = nil
                    end
                    camera.CameraType = Enum.CameraType.Custom
                    isViewing = false
                end
            end)
        end
    end
end

-- Function to stop viewing
local function unviewPlayer()
    if renderConnection then
        renderConnection:Disconnect()
        renderConnection = nil
    end
    camera.CameraType = Enum.CameraType.Custom
    isViewing = false
end

-- Toggle minimize state with animation
local function toggleMinimize()
    isMinimized = not isMinimized
    
    if isMinimized then
        -- Animate to minimized state (only title bar visible)
        local tween = TweenService:Create(
            mainFrame,
            TweenInfo.new(0.3, Enum.EasingStyle.Quad),
            {Size = UDim2.new(mainFrame.Size.X.Scale, mainFrame.Size.X.Offset, 0, 30)}
        )
        tween:Play()
        minimizeButton.Text = "+"
    else
        -- Animate to expanded state
        local tween = TweenService:Create(
            mainFrame,
            TweenInfo.new(0.3, Enum.EasingStyle.Quad),
            {Size = UDim2.new(0, 300, 0, 400)}
        )
        tween:Play()
        minimizeButton.Text = "_"
    end
end

-- Handle camera rotation while viewing
UserInputService.InputChanged:Connect(function(input)
    if isViewing and input.UserInputType == Enum.UserInputType.MouseMovement and UserInputService:IsMouseButtonPressed(Enum.UserInputType.MouseButton2) then
        cameraAngleY = cameraAngleY - input.Delta.X * 0.2
        cameraAngleX = math.clamp(cameraAngleX - input.Delta.Y * 0.2, -80, 80)
    end
end)

-- Set up buttons
viewButton.MouseButton1Click:Connect(function()
    if selectedPlayer then
        viewPlayer(selectedPlayer)
    end
end)

unviewButton.MouseButton1Click:Connect(function()
    unviewPlayer()
end)

closeButton.MouseButton1Click:Connect(function()
    screenGui:Destroy()
    unviewPlayer()
end)

minimizeButton.MouseButton1Click:Connect(toggleMinimize)

-- Update list when player joins/leaves
Players.PlayerAdded:Connect(updatePlayerList)
Players.PlayerRemoving:Connect(updatePlayerList)

-- Initial player list update
updatePlayerList()

-- Button hover effects
local function setupButtonHover(button, normalColor, hoverColor)
    button.MouseEnter:Connect(function()
        button.BackgroundColor3 = hoverColor
    end)
    button.MouseLeave:Connect(function()
        button.BackgroundColor3 = normalColor
    end)
end

setupButtonHover(viewButton, Color3.fromRGB(60, 120, 60), Color3.fromRGB(80, 160, 80))
setupButtonHover(unviewButton, Color3.fromRGB(120, 60, 60), Color3.fromRGB(160, 80, 80))
setupButtonHover(closeButton, Color3.fromRGB(200, 50, 50), Color3.fromRGB(255, 80, 80))
setupButtonHover(minimizeButton, Color3.fromRGB(100, 100, 100), Color3.fromRGB(150, 150, 150))
