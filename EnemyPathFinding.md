# Planet-Rising
#===============================================================================
# Enemy AI-Pathfinding Algorithm [A*] 
# By Santiphap Butrmaeklong   57070503437
#===============================================================================
#
# Modifications: 
# Goal location may now be inpassable, pathfinder will still reach it.
# Pathfinder still works "as intended" when the move route is set to repeat.
# Added parameter: distance (explained below).
# Note: Pathfinder is a bit more processor heavy when used with a repeating 
# move route (it was useless before, so this is only a benefit)
#===============================================================================

=begin

find_path(x, y, ev = 0, wait = false, distance = 0)

=end

module Jet
  module Pathfinder
    
    # allow the
    # pathfinder more time to find a path. 1000 is default, as it is enough for
    # a 100x100 MAZE
    MAXIMUM_ITERATIONS = 1000
    
  end
end

class Node
  
  include Comparable

  attr_accessor :point, :parent, :cost, :cost_estimated

  def initialize(point)
    @point = point
    @cost = 0
    @cost_estimated = 0
    @on_path = false
    @parent = nil
  end

  def mark_path
    @on_path = true
    @parent.mark_path if @parent
  end
   
  def total_cost
    cost + cost_estimated
  end

  def <=>(other)
    total_cost <=> other.total_cost
  end
   
  def ==(other)
    point == other.point
  end
end

class Point
  
  attr_accessor :x, :y
  
  def initialize(x, y)
    @x, @y = x, y
  end

  def ==(other)
    return false unless Point === other
    @x == other.x && @y == other.y
  end

  def distance(other)
    (@x - other.x).abs + (@y - other.y).abs
  end

  def relative(xr, yr)
    Point.new(x + xr, y + yr)
  end
end

class Game_Map
  
  def each_neighbor(node, char = $game_player)
    x = node.point.x
    y = node.point.y
    nodes = []
    4.times {|i|
      i += 1
      new_x = round_x_with_direction(x, i * 2)
      new_y = round_y_with_direction(y, i * 2)
      next unless char.passable?(x, y, i * 2)
      nodes.push(Node.new(Point.new(new_x, new_y)))
    }
    nodes
  end
  
  #(added distance parameter)
  def find_path(tx, ty, sx, sy, dist, char = $game_player)
    start = Node.new(Point.new(sx, sy))
    goal = Node.new(Point.new(tx, ty))
    #(added distance handling)
    return [] if start == goal or (dist > 0 and start.point.distance(goal.point) <= dist)
    return [] if ![2, 4, 6, 8].any? {|i| char.passable?(tx, ty, i) }
    open_set = [start]
    closed_set = []
    path = []
    iterations = 0
    loop do
      return [] if iterations == Jet::Pathfinder::MAXIMUM_ITERATIONS
      iterations += 1
      current = open_set.min
      return [] unless current
      each_neighbor(current, char).each {|node|
        #(added distance handling)
        if node == goal or (dist > 0 and node.point.distance(goal.point) <= dist)
          node.parent = current
          node.mark_path
          return recreate_path(node)
        end
        next if closed_set.include?(node)
        cost = current.cost + 1
        if open_set.include?(node)
          if cost < node.cost
            node.parent = current
            node.cost = cost
          end
        else
          open_set << node
          node.parent = current
          node.cost = cost
          node.cost_estimated = node.point.distance(goal.point)
        end
      }
      closed_set << open_set.delete(current)
    end
  end
  
  def recreate_path(node)
    path = []
    hash = {[1, 0] => 6, [-1, 0] => 4, [0, 1] => 2, [0, -1] => 8}
    until node.nil?
      pos = node.point
      node = node.parent
      next if node.nil?
      ar = [pos.x <=> node.point.x, pos.y <=> node.point.y]
      path.push(RPG::MoveCommand.new(hash[ar] / 2))
    end
    return path
  end
end

class Game_Character
  
  #(added handling for repeated move route (recalculates path 
  # each step so it doesn't just loop it's old path route and will revalidate  
  # if x or y changes, will follow variable value if x and y are set to it)
  def find_path(x, y, dist = 0)
    path = $game_map.find_path(x, y, self.x, self.y, dist).reverse
    if !@move_route.repeat
      @move_route.list.delete_at(@move_route_index)
      @move_route.list.insert(@move_route_index, *path)
      @move_route_index -= 1
    elsif path.length > 0
      process_move_command(path[0])
      @move_route_index -= 1
    end
    
  end
end

class Game_Interpreter
  
  #(added distance parameter)
  def find_path(x, y, ev = 0, wait = false, dist = 0)
    char = get_character(ev)
    #(added distance parameter)
    path = $game_map.find_path(x, y, char.x, char.y, dist)
    path.reverse!
    path.push(RPG::MoveCommand.new(0))
    route = RPG::MoveRoute.new
    route.list = path
    route.wait = wait
    route.skippable = true
    route.repeat = false
    char.force_move_route(route)
  end
  
end
