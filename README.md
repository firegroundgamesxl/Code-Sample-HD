-- SERVICES
local replicated_storage = game:GetService("ReplicatedStorage")
local players = game:GetService("Players")
local run_service = game:GetService("RunService")

-- INSTANCES
local player = players.LocalPlayer
local camera = workspace.CurrentCamera
local assets = replicated_storage.assets
local modules = replicated_storage.modules
local data_modules = modules.data
local utility_modules = modules.utility
local packages = modules.packages
local controllers = modules.controllers

-- MODULES
local forge_vfx = require(packages.forge_vfx)
local camera_controller = require(controllers:WaitForChild("camera_controller"))
local trove = require(utility_modules:WaitForChild("trove"))
local types = require(data_modules:WaitForChild("replicated_types"))
local vfx_actions = require(utility_modules:WaitForChild("vfx"))
local sound_manager = require(utility_modules:WaitForChild("sound_manager"))
local toolbox = require(modules.utility.toolbox)
local character_actions = require(modules.utility.character_actions)

-- VARIABLES
local client_flying_kick = {}
client_flying_kick._troves = {}
local flying_kick_assets = assets.characters.shinra.combat.skills.base.flying_kick
local grounded_assets = flying_kick_assets.grounded
local aerial_assets = flying_kick_assets.aerial
local flying_kick_grounded_animations = grounded_assets.animations
local flying_kick_aerial_animations = aerial_assets.animations
local dash_animation = flying_kick_grounded_animations.dash
local loop_animation = flying_kick_grounded_animations.loop
local recovery_animation = flying_kick_grounded_animations.recovery
local flying_kick_grounded_vfx = grounded_assets.vfx
local flying_kick_aerial_vfx = aerial_assets.vfx

local active_trove = nil
local active_vfx = {}

local function play_keyframe_properties(refs, props, particles, fps)
	fps = fps or 60

	for ref_name, prop_table in pairs(props) do
		local ref = refs[ref_name]
		if not ref then continue end

		for prop_name, keyframes in pairs(prop_table) do
			local frames = {}
			for k, _ in pairs(keyframes) do
				if typeof(k) == "number" then
					table.insert(frames, k)
				end
			end
			table.sort(frames)

			if not table.find(frames, 0) and keyframes.default then
				table.insert(frames, 1, 0)
				keyframes[0] = keyframes.default
			end

			for i = 1, #frames - 1 do
				local f0, f1 = frames[i], frames[i+1]
				local v0, v1 = keyframes[f0], keyframes[f1]
				local t0, t1 = f0/fps, f1/fps
				local duration = t1 - t0

				task.delay(t0, function()
					local tween_info = TweenInfo.new(duration, Enum.EasingStyle.Linear)
					local goal = {}
					goal[prop_name] = v1
					local tween = game:GetService("TweenService"):Create(ref, tween_info, goal)
					tween:Play()
				end)
			end

			if keyframes.default and #frames == 0 then
				ref[prop_name] = keyframes.default
			end
		end
	end

	if particles then
		for frame, particle_data in pairs(particles) do
			local t = frame / fps
			task.delay(t, function()
				local ref = particle_data.reference
				local action = particle_data.action

				if action == "emit" then
					vfx_actions.emit(ref)
				elseif action == "enable_particle" then
					vfx_actions.enable_particle(ref)
				elseif action == "disable_particle" then
					vfx_actions.disable_particle(ref)
				end
			end)
		end
	end
end

function client_flying_kick:cleanup()
	for _, vfx_instance in active_vfx do
		vfx_actions.destroy(vfx_instance)
	end
	active_vfx = {}

	if active_trove then
		active_trove:Destroy()
		active_trove = nil
	end
end

function client_flying_kick:start_grounded(data: types.flying_kick)
	self:cleanup()

	active_trove = trove.new()

	local attacker = data.attacker
	local attacker_humanoid = attacker:FindFirstChild("Humanoid") :: Humanoid
	local attacker_root_part = attacker:FindFirstChild("HumanoidRootPart")
	local attacker_animator = attacker_humanoid and attacker_humanoid:FindFirstChild("Animator") :: Animator

	local dash_track = attacker_animator:LoadAnimation(dash_animation)
	active_trove:Add(dash_track, "Stop")
	dash_track:Play()

	task.delay(dash_track.Length, function()
		if not active_trove then return end
		local loop_track = attacker_animator:LoadAnimation(loop_animation)
		active_trove:Add(loop_track, "Stop")
		loop_track:Play()
	end)

	task.delay(0.5, function()
		if not active_trove then return end
		local result = vfx_actions.spawn({
			vfx_instance = flying_kick_grounded_vfx.fxpart,
			parent = attacker_root_part,
		})
		if result then
			local fxpart = result.Instance
			fxpart.CFrame = attacker_root_part.CFrame * CFrame.new(0, 0, -3) * CFrame.Angles(0, math.rad(90), 0)
			local weld = Instance.new("WeldConstraint")
			weld.Part0 = attacker_root_part
			weld.Part1 = fxpart
			weld.Parent = fxpart
			table.insert(active_vfx, fxpart)
		end
	end)
end

function client_flying_kick:stop_grounded(data: types.flying_kick)
	self:cleanup()

	active_trove = trove.new()

	local attacker = data.attacker
	local attacker_humanoid = attacker:FindFirstChild("Humanoid") :: Humanoid
	local attacker_root_part = attacker:FindFirstChild("HumanoidRootPart")
	local attacker_animator = attacker_humanoid and attacker_humanoid:FindFirstChild("Animator") :: Animator

	local recovery_track = attacker_animator:LoadAnimation(recovery_animation)
	active_trove:Add(recovery_track, "Stop")
	recovery_track:Play()

	recovery_track.Stopped:Once(function()
		self:cleanup()
	end)
end

function client_flying_kick:aerial_swing(data: types.sweepy_sweep)
	local attacker = data.attacker
	local attacker_humanoid = attacker:FindFirstChild("Humanoid") :: Humanoid
	local attacker_root_part = attacker:FindFirstChild("HumanoidRootPart")
	local attacker_animator = attacker_humanoid and attacker_humanoid:FindFirstChild("Animator") :: Animator

	local active_trove = trove.new()
	self._troves[attacker] = active_trove

	if attacker_animator then
		local track = attacker_animator:LoadAnimation(flying_kick_aerial_animations.jump)
		active_trove:Add(track)

		local stopped = track.Stopped:Once(function()
			track = attacker_animator:LoadAnimation(flying_kick_aerial_animations.swing)
			track:Play()
			track:GetMarkerReachedSignal("freeze"):Once(function()
				track:AdjustSpeed(0)
			end)
		end)

		active_trove:Add(stopped)
		track:Play()

		task.delay(0.2, function()
			track:Stop(0.1)
		end)
	end

	local jump_result = vfx_actions.spawn({
		vfx_instance = flying_kick_aerial_vfx.jump_part,
		parent = workspace.effects,
	})
	local left_leg_result = vfx_actions.spawn({
		vfx_instance = flying_kick_aerial_vfx.left_leg,
		parent = attacker["Left Leg"],
	})
	local upper_kick_result = vfx_actions.spawn({
		vfx_instance = flying_kick_aerial_vfx.upperkick,
		parent = attacker_root_part,
	})

	local jump_part = jump_result and jump_result.Instance
	local left_leg_flames = left_leg_result and left_leg_result.Instance
	local upper_kick = upper_kick_result and upper_kick_result.Instance

	active_trove:Add(function()
		if jump_part then vfx_actions.destroy(jump_part) end
		if left_leg_flames then vfx_actions.destroy(left_leg_flames) end
		if upper_kick then vfx_actions.destroy(upper_kick) end
	end)

	local references = {
		jumpmesh1 = jump_part and jump_part:FindFirstChild("jumpmesh1"),
		jumpmesh2 = jump_part and jump_part:FindFirstChild("jumpmesh2"),
		jumpweld1 = jump_part and jump_part:FindFirstChild("jumpweld1"),
		jumpweld2 = jump_part and jump_part:FindFirstChild("jumpweld2"),
		PointLight = left_leg_flames and left_leg_flames:FindFirstChild("PointLight"),
	}

	task.delay(0.216, function()
		if jump_part and jump_part.Parent then
			jump_part.CFrame = attacker_root_part.CFrame
		end
	end)

	local properties = {
		jumpmesh1 = {
			Transparency = {
				[12] = 1, [50] = 1, [0] = 1, [13] = 0.75
			},
			Size = {
				[0] = Vector3.new(9.553000450134277, 12.640000343322754, 12.640000343322754),
				[50] = Vector3.new(6, 22, 22),
				default = Vector3.new(9.553000450134277, 12.640000343322754, 12.640000343322754),
				[13] = Vector3.new(15, 12, 12)
			}
		},
		jumpweld1 = {
			C0 = {
				[0] = CFrame.new(0.061981201171875, 0.021399497985839844, -0.141998291015625, 0, -8.742277657347586e-08, 1, 1, 0, 0, 0, 1, 8.742277657347586e-08),
				default = CFrame.new(-407.05804443359375, 9.121392250061035, -205.63099670410156, 0, 0, 1, 1, 0, 0, 0, 1, 0),
				[25] = CFrame.new(0.061981201171875, -0.14960575103759766, -0.141998291015625, 0, 0.34519368410110474, 0.9385315179824829, 1, 0, 0, 0, 0.9385315179824829, -0.34519368410110474),
				[50] = CFrame.new(0.061981201171875, -2.122035026550293, -0.141998291015625, 0, 0.7765404582023621, 0.6300674080848694, 1, 0, 0, 0, 0.6300674080848694, -0.7765404582023621),
				[13] = CFrame.new(0.061981201171875, 3.365309715270996, -0.141998291015625, 0, -8.742277657347586e-08, 1, 1, 0, 0, 0, 1, 8.742277657347586e-08)
			},
		},
		jumpmesh2 = {
			Transparency = {
				[12] = 1, [40] = 1, [0] = 1, [13] = 0.75
			},
			Size = {
				[0] = Vector3.new(9.035000801086426, 2.245000123977661, 8.951000213623047),
				[40] = Vector3.new(25, 5, 25),
				default = Vector3.new(9.035000801086426, 2.245000123977661, 8.951000213623047),
				[13] = Vector3.new(9.035000801086426, 2.245000123977661, 8.951000213623047)
			}
		},
		jumpweld2 = {
			C0 = {
				[0] = CFrame.new(-0.23785400390625, -1.4881000518798828, -0.0744476318359375, 8.742277657347586e-08, 0, 1, 0, 1, 0, -1, 0, 8.742277657347586e-08),
				[40] = CFrame.new(-0.23785400390625, -1.4881000518798828, -0.0744476318359375, 0.50156170129776, 0, 0.8651218414306641, 0, 1, 0, -0.8651218414306641, 0, 0.50156170129776),
				default = CFrame.new(-407.3578796386719, 7.6118927001953125, -205.56344604492188, 0, 0, 1, 0, 1, -0, -1, 0, 0),
				[13] = CFrame.new(-0.23785400390625, -1.4881000518798828, -0.0744476318359375, 8.742277657347586e-08, 0, 1, 0, 1, 0, -1, 0, 8.742277657347586e-08)
			},
		},
		PointLight = {
			Enabled = {
				[0] = false, default = true, [14] = false, [15] = true, [89] = false,
			},
			Brightness = {
				[0] = 5, [20] = 7, default = 5, [15] = 1
			}
		}
	}

	local particles = {
		[12] = { action = "emit", reference = jump_part },
		[13] = { action = "emit", reference = upper_kick },
		[15] = { action = "enable_particle", reference = left_leg_flames },
		[89] = { action = "disable_particle", reference = left_leg_flames },
	}

	local frame_skip = 0
	for name, prop_data in properties do
		for property, keyframes in prop_data do
			local shifted = {}
			for keyframe, value in pairs(keyframes) do
				if keyframe == "default" then
					shifted.default = value
				elseif typeof(keyframe) == "number" then
					shifted[keyframe - frame_skip] = value
				end
			end
			prop_data[property] = shifted
		end
	end

	play_keyframe_properties(references, properties, particles, 60)
end

function client_flying_kick:aerial_hit(data: types.sweepy_sweep)
	local attacker = data.attacker
	local attacker_humanoid = attacker:FindFirstChild("Humanoid") :: Humanoid
	local attacker_animator = attacker_humanoid and attacker_humanoid:FindFirstChild("Animator") :: Animator

	local active_trove = trove.new()
	self._troves[attacker] = active_trove

	if attacker_animator then
		local track = attacker_animator:LoadAnimation(flying_kick_aerial_animations.hit)
		active_trove:Add(track)
		track:Play()
	end
end

function client_flying_kick:aerial_land(data: types.sweepy_sweep)
	local attacker = data.attacker
	local attacker_humanoid = attacker:FindFirstChild("Humanoid") :: Humanoid
	local attacker_root_part = attacker:FindFirstChild("HumanoidRootPart")
	local attacker_animator = attacker_humanoid and attacker_humanoid:FindFirstChild("Animator") :: Animator

	local active_trove = trove.new()
	self._troves[attacker] = active_trove

	local params = RaycastParams.new()
	params.FilterType = Enum.RaycastFilterType.Include
	params.FilterDescendantsInstances = {workspace.map}

	local result = workspace:Raycast(attacker_root_part.Position, Vector3.new(0, -5, 0), params)
	local cframe = result and CFrame.new(result.Position) or CFrame.new(attacker_root_part.Position + Vector3.new(0, -2.97, 0))

	vfx_actions.spawn({
		vfx_instance = flying_kick_aerial_vfx.explosion,
		parent = workspace.effects,
		attach = cframe
	})

	if not data.hit_someone and attacker_animator then
		local track = attacker_animator:LoadAnimation(flying_kick_aerial_animations.land)
		track:Play()
		active_trove:Add(track)
	end
end

function client_flying_kick:handle_grounded(data)
	if data.pressed then
		self:start_grounded(data)
	else
		self:stop_grounded(data)
	end
end

function client_flying_kick:handle_aerial(data)
	if not data.state then return end
	local action = self["aerial_" .. data.state]
	if action then
		action(self, data)
	end
end

function client_flying_kick:execute(data: types.flying_kick)
	if not data.attacker then return end

	if data.cleanup then
		self:cleanup()
		return
	end

	if data.aerial then
		self:handle_aerial(data)
	else
		self:handle_grounded(data)
	end
end

return client_flying_kick
